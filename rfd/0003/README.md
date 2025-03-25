---
authors: David Mak, Kristopher Lam
state: prediscussion
discussion:
labels: direction, infrastructure
---

# Architecture for Grader Workers

The goal of this RFD is to established the expected architecture for the grader worker. The grader worker is defined as the executable responsible for running one or more commands as defined by the grading configuration and within an isolated environment.

## Background

In ZINC 1.0 (hereinafter referred as 1.0), the grader executable consists of a monolith which performs both control plane duties, i.e. receiving grader jobs from IPC and dispatching jobs to workers for execution, as well as worker duties, i.e. the formulation of execution commands, collection of executable outputs, and aggregation into various reports for user viewing. 

### Grader Worker 1.0 Process

TODO

### Grader Worker 1.0 Design Drawbacks

While this design has served well throughout the lifetime of 1.0, there are several shortcomings to this approach.

1. Difficulty in Partial Recovery and Rehydration

One of the design decisions made during the development of 1.0 is that the grader should be stateless, such that persistence can be implemented by delegating to an external system and thus the grader only needs to be concerned with saving and restoring backup states. However, since only the control plane has direct access to the system holding persistence states, a partially graded pipeline cannot be reused during rehydration, forcing the re-execution of the entire grading pipeline.

2. Limited Control over Isolation Environment

The original grader utilizes the following procedure when executing a single pipeline stage:

- Formulate the command used for executing the pipeline stage
- Send a command creating the isolation environment with parameters configured from the grading configuration, lazily creating an image template for caching
- Send a command starting the isolation environment with the command
- Wait for the isolation environment to complete execution
- Read-back files from the container that are required for the grading and/or scoring process
- Destroy the isolation environment, retaining any files that may be needed in a later grading stage
- Parse the files to create a stage report and/or score report

Since the grader executable lives outside of the isolation environment and can only interface with the container environment via the use of shell commands and limited mechanisms defined by the isolation environment (e.g. `docker cp` for copying from/to-host for Docker), this limits the methods we can manipulate the isolation environment.

3. Inflexible Scoring Strategies

Scoring strategies are currently defined by the grading executable, and each stage defines their own supported scoring strategies. However, we have received requests over the years by various teaching staff to introduce more flexible scoring strategies, such that by the amount of difference between the expected output and actual output, or requiring several test cases to all pass in order to give credit to the test case.

In order to support these new scoring strategies, Grader 1.0 will require code-level changes to 1. add support for these strategies, and 2. implement and allowlist these strategies for existing stages, which introduces a large overhead.

4. Unable to Mix Isolation Environments

In Grader 1.0, while the pipeline design allows for the mixing of stages that utilize isolation environments and stages that can be executed locally on the host of the grader executable, there is no mechanism to mix stages that utilize different isolation environments, nor is it possible to mix stages that utilize the same isolation environment but requiring different hardware capabilities.

Consider a scenario where we have two machines available for dispatch, one with GPUs and one without, and given we have a pipeline that contains some stages requiring the use of GPUs and other stages only requiring the CPU. In Grader 1.0, since pipelines are bound to a single Docker Engine throughout the execution of the pipeline, all jobs will be dispatched to the host with GPU support, making the CPU-only host underutilized.

## Proposal

We propose the address the above issues by splitting the grader into two components - one responsible for managing the control plane that executes in the host system, and one executing within the isolation environment acting as a driver for one or more pipeline stages. These two components are connected via an external system acting as the orchestrator, and a network object store will be added to facilitate file storage across the control plane and different workers. This approach has several advantages:

- By splitting the grading process into two components, the error handling process is greatly simplified as the orchestrator can help save and propagate errors in a manner without unintentionally bringing the entire system down.
- The usage of an external system as an orchestrator avoids the need to implement our own control plane logic, while also adding support for resource monitoring and auto-scaling.
- The addition of the object store enables the storage of partial grading results and reports, which in turn allows the sharing of these resources across different hosts as well as simplifies rehydration.

**This RFD will focus on describing the architecture for the grader workers. The high-level description of the architecture for the control plane is described in RFD 0004.**

### High-Level Control Flow

The grader worker will be responsible for performing the following two high-level actions.

#### Environment Setup Workflow

The environment setup workflow executes a set of commands used to create a template for the grading environment, such that grading workflows can derive an already-setup environment instead of having to re-setup all prerequisite packages. This is triggered when a grading configuration is updated for caching purposes.

1. An isolation environment will be created for each host by the control plane, and the grader worker will start executing within the isolation environment.
2. The grader worker will fetch the grading configuration it is assigned to from the object store.
3. For each pipeline stage definition in the grading configuration, the grader worker will configure the environment by installing necessary packages and running pre-execution commands. If the host of the isolation environment does not support any capabilities required by this stage definition, break out of the loop.
4. Use the environment-specific method to generate a template for the grading environment, e.g. `docker commit` for Docker.
5. If there are no remaining pipeline stages to generate an environment template for, exit the workflow. Otherwise, callback to the orchestrator with the remaining stages.

#### Grading Workflow

The grading workflow executes one or more grading stages, and is triggered when a submission is initially received by the control plane, or when an ongoing grading workflow cannot continue on the allocated grader worker and needs to be re-dispatched to another worker with supported capabilities.

1. An isolation environment will be created by the control plane, and the grader worker will start executing within the isolation environment.
2. The grader worker will fetch the grading configuration it is assigned to from the object store, and parse the first pipeline stage definition.
3. The grader worker will verify that the isolation environment contains the capability to execute the pipeline stage definition. If not, the grader worker will callback to the orchestrator indicating the capabilities required for the stage definition, and the grader worker will terminate.
4. The grader worker will fetch the necessary files for grading, including the student submission, helper files uploaded by the teaching staff, and any intermediate files generated during the execution of previous stages.
5. The grader worker will execute one or more commands that performs the task(s) as specified by the parsed pipeline stage definition. The worker will monitor the `stdout`/`stderr` streams and store into a memory buffer.
6. The grader worker will read any additional files that are necessary for the generation of the stage report, such as serialized test output results.
7. The grader worker will generate a stage report containing all execution information of this stage, and upload this report into the object store.
8. Repeat Step 2 until the grading pipeline is empty, or the current isolation environment instance does not have the capability to execute a pipeline stage (see Step 3).
9. If there are remaining pipeline stages to be executed, the grader worker will package the working directory of the isolation environment and upload this into the object store.
10. The grader worker will callback to the orchestrator with the remaining stages of this pipeline, as well as any context required for subsequent execution of the pipeline.

## Implementation

First Draft (in Kotlin):

```kotlin
fun workflow(gradingConfig: GradingConfig) {
	if (gradingConfig.isEmpty()) {
		temporal.sendGradingComplete()
	}

	// TODO: How to join a divergent workflow? This is necessary for stages such as report aggregation

	val currentStage = gradingConfig.popFront()

	// do grading stuff...

	for (dependents in currentStage.dependents) {
		launch {
			temporal.sendGradeWorkflow(dependents.getDependencyChain())
		}
	}
}
```

Need to think about how to resolve forward dependencies

## Problems To Be Decided On

- Should isolation environments eagerly setup as many pipeline stages as the dependency chain allows for?
	- Benefits
		- Enables multiple stages to be executed within the same environment, see below
		- Few images means smaller image footprints
	- Drawbacks
		- Consider a mixed pipeline: `A -> B[cap=GPU] -> C`: GPU-capable hosts will need to generate environments for `ABC` and `BC`, since `A` may be executed on a non-GPU-capable host
		- There may be a risk that sequentially executing the setup logic for more than one stage may have conflicts or incompatibilities
- Should we execute as many pipeline stages as the dependency chain allows for?
	- Benefits
		- Reduce overhead of needing to download/upload environment to object store
	- Drawbacks
		- (None?)
- If we want to eventually support fork-join-like evaluation, how do we resolve Phi-like join points?
	- Add an activity that checks the number of uploaded reports, and require that number to match the number of forked sub-workflows? (Similar to how `waitcnt` works in RDNA GPUs)