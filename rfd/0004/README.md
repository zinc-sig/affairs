---
authors: David Mak, Kristopher Lam
state: prediscussion
discussion:
labels: direction, infrastructure
---

# Architecture for Control Plane

The aim of this document is to establish the expected architecture for the control plane. The control plane is defined as the executable responsible for receiving requests from the frontend, and creating isolation environments for grader workers to execute in.

## Background

TODO: Copy from RFD 0003

## Proposal

TODO: Copy from RFD 0003

**This RFD will focus on describing the architecture for the Grader Control Plane. The high-level description of the architecture for the Grader Worker is described in RFD 0003.**

### High-Level Control Flow

The control plane will be responsible for performing the following high-level actions.

### Grading Configuration Validation Workflow

This workflow performs validation on a grading configuration, and is triggered when the frontend sends a request for such.

1. The control plane will fetch the grading configuration from either the RPC payload or the object store, as specified by the RPC payload itself.
2. The control plane will parse the grading configuration into an implementation-defined format (see RFD 0006).
3. The control plane will validate the implementation-defined format in an implementation-defined manner, such as:
	- Verifying that all helper files specified in the grading configuration are provided
	- Verifying that any constraints imposed by the grading configuration specification is not violated
	- Verifying that, for every stage that will be generated from the grading configuration, there is at least one isolation environment host that is capable of running the stage
4. A list of diagnostics will be generated from the validation process, and the validation result is returned to the frontend.

### Environment Setup Workflow

This workflow sets up one or more templates for the isolation environment by creating an appropriate isolation environment and passing the grading configuration for the grader worker to perform its environment setup workflow (see RFD 0003). This is triggered when the frontend sends a request for such.

1. The control plane will fetch the grading configuration from the object store.
2. The control plane will parse the grading configuration into an implementation-defined format.
3. The control plane will validate the implementation-defined grading configuration as if by executing Step 3 of the [[#Grading Configuration Validation Workflow]], raising an error if the configuration format fails to validate.
4. The control plane will create an initial isolation environment on every connected host, and pass the grading configuration to all hosts for setup.

### Grading Workflow

This workflow executes the grading workflow for one or more submissions by creating an isolation environment from an already-created environment template, and passing the grading configuration for the grader worker to perform its grading workflow (see RFD 0003). This is triggered when the frontend sends a request for such.

1. The control plane will fetch the grading configuration from the object store.
2. The control plane will parse the grading configuration into an implementation-defined format.
3. The control plane will validate the implementation-defined grading configuration as if by executing Step 3 of the [[#Grading Configuration Validation Workflow]], raising an error if the configuration format fails to validate.
4. For each submission in the RPC payload, the control plane will enqueue the submission payload to the orchestrator. The orchestrator will then dispatch the payload to an isolation environment host, and start executing the grading workflow.

## Problems To be Decided On

- Should the parsing of grading configuration be the responsibility of the control plane?
- Should the aggregation of stage reports be the responsibility of the control plane?
- Does the control plane also need to perform `fileStructureValidation`/`diffWithSkeleton`?
- If an isolation environment host is connected to the orchestrator after the grading configuration is created, should we provide a mechanism for those late-initialized hosts to lazily create environment templates?
	- How do we verify that a given environment template exists on a given host?