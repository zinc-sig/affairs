---
authors: David Mak, Kristopher Lam
state: committed
discussion: #2
labels: process
---

# Requests for Discussion (RFD)

The aim of this document is to lay down the foundations on how to draft Requests for Discussions (RFDs), and the process of submitting and formalizing RFDs in the ZINC Special Interest Group (SIG). Our goal of adapting RFDs into the engineering process of the ZINC project is to ensure that all features proposed to be integrated into the ZINC project are sufficiently vetted by different stakeholders, such that the implementation of these features addresses the needs and/or concerns of all stakeholders.

This document is loosely based on [Oxide Computer Company's RFD 1](https://rfd.shared.oxide.computer/rfd/0001) (hereinafter referred to as *OCC rfd1*).

## 1. When to use an RFD

In general, RFDs should be used whenever, upon the implementation of an idea, the [Semantic Versioning Specification (SemVer)](https://semver.org/) dictates that the major or minor version of the project implementing said idea must be incremented. These include, but are not limited to, the following circumstances:

- Architectural decisions, such as the implementation or usage of a new software, in-place of or in addition to the current architectural software stack
- Changes that are visible to users of the system and modifies their workflow respectively
- Changes to the internal API, where APIs that have been changed modifies the system in a novel way, e.g. implementation of a new feature or rewriting of a feature
- Changes to internal processes

### 1.1 Insights to include in RFDs

Due to the nature of RFDs, RFDs should be evaluated by different sets of insights that are more relevant to the topic at hand. A list of starting points is provided below:

- Explain why such a feature or change is necessary or beneficial
	- Refer to issues or other discussions if relevant
	- List out the existing option(s) (if any) and explain why it is inadequate
- Discuss any alternative options (if any), their merits and drawbacks
- List out the prerequisites (if any), the steps towards implementing the idea, and any post-implementation verification
	- Give a timeline for the expected progress and/or completion of the idea

### 1.2 Attention to Pay at when writing about concrete implementation

Unless there are no other viable existing alternatives, refrain from starting a RFD with a specific solution or using a particular framework. Instead, focus on providing a high-level overview of the problem and potential solutions.

Optimally we would like to be as objective as possible when discussing potential solutions. We should avoid making assumptions about the implementation details or the specific technologies that will be used. We should also consider the trade-offs between different approaches and weigh the pros and cons of each option.

Experimentation is encouraged to provide evidence based review on the feasibility and effectiveness of the proposed solution(s).

## 2. RFD Metadata and State

We adopt the usage of metadata and state in RFDs as per [*OCC rfd1* §2](https://rfd.shared.oxide.computer/rfd/0001#_rfd_metadata_and_state). The details are summarized as follows.

### 2.1 RFD Metadata

All RFDs must contain metadata in the top of the Markdown document defining the RFD. The following metadata must be present:

- `authors`: The list of authors of an RFD.
- `state`: One of the states listed below.
- `discussion`: Link to the Pull Request integrating this RFD; usually an issue number.
- `labels`: Command-separated list of text labels that specify the categories that the RFD falls into.

### 2.2 RFD States

RFDs fall into one of the following six states.

1. `prediscussion`: The RFD is actively under the drafting process and may be subject to major revision.
2. `ideation`: The RFD has been assigned to a simple problem description or statement, but there is currently no active work on it.
3. `discussion`: The RFD is submitted as a PR and is open for discussion.
4. `published`: The RFD is agreed upon and is merged (or in the process of).
5. `committed`: The RFD is entirely implemented.
6. `abandoned`: The RFD is not viable or should otherwise be ignored.

## 3. RFD lifecycle

This section will describe how RFDs are created, submitted for review, and eventually adopted into the project.

### 3.1 Reserve an RFD number

You should first reserve an RFD number. This can be done by inspecting the RFD number of the greatest value, and taking the next number after. This ensures that all RFDs are numerically sequential.

#### 3.1.1 Create a branch for your RFD

Create a new git branch in this repository for your RFD by naming it `rfd/<rfd_number>`. Note that the RFD number should be prefixed with zeros if the RFD number is less than 4 digits.

```
$ git checkout -b rfd/0042
```

#### 3.1.2 Create a placeholder RFD

Create a placeholder RFD by creating a directory corresponding to your RFD in the `rfd` directory, and copying the RFD template.

```
$ mkdir -p rfd/0042
$ cp rfd/templates/prototype.md rfd/0042/README.md
```

Fill in the RFD metadata and add your name as an author. The status of the RFD should be `ideation` or `prediscussion`, depending on your ability to continue working on the RFD at this point.

#### 3.1.3 Push your RFD branch remotely

Push your changes of your RFD branch to the `affairs` repository.

```
$ git add rfd/0042/README.md
$ git commit -m 'rfd/0042: Add placeholder for RFD <Title>'
$ git push -u origin rfd/0042
```

#### 3.1.4 Iterate on your RFD in your branch

Start working on the RFD! A template is available in [`rfd/templates`](../rfd/templates).

There are no hard requirements to squash your commits - do this at your own discretion.

### 3.2 Discuss your RFD

Once you are ready to receive feedback for your RFD, make sure that all local changes are pushed to the remote branch. Be sure to also change the state to `discussion` or `ideation` before creating the PR.

#### 3.2.1 Push your RFD branch remotely

```
$ git commit -am 'rfd/0042: Add RFD for <Title>'
$ git push
```

#### 3.2.2 Open a Pull Request

Open a pull request on GitHub to merge your branch into `main`. You will need to push another change to this branch to update the `discussion` metadata of the RFD.

#### 3.2.3 Discuss the RFD on the Pull Request

The final authority over whose comments to accept lie solely with the maintainers of the ZINC project. However, it is imperative that all discussions should be constructive.

#### 3.2.4 Merge the Pull Request

After the Pull Request of the RFD is opened for one week, the RFD can be merged into `main` and its state changed from `discussion` to `published`. Try to ensure that the RFD is at least read by one other person.

Discussions can continued on `published` RFDs by using the `discussion` link in the RFD metadata.

#### 3.2.5 Making changes to an RFD

Changes may be made even after an RFD is moved to `published` state.

To do so, create a new branch (e.g. `rfd/0001/some-modification`) for each logical change in the RFD, and commit all relevant changes. Afterwards, push the branch onto GitHub, create a Pull Request with the original authors as reviewers, and follow the procedure as outlined previously for new RFDs.

#### 3.3 Committing to an RFD

Once an RFD has been implemented, its state should be moved to `committed` by the same process as outlined in the previous section.

## 4. Changing the RFD process

This document is also an RFD, so it is also subject to the same amendment processes as outlined [here][[#3.2.5 Making changes to an RFD]].

## 5. Tooling

### 5.1 Labels

We adopt the following canonical labels to categorize the scope of various RFDs.

- `direction`: Internal information, organization-related, or governance.
- `process`: Related to a process of some kind.
- `infrastructure`: self-explanatory
- `deployment`: self-explanatory
- `security`: all aspects of security
- `metrics`: collected metrics
- `debug`: debugging
- `interop`: interoperability with external systems
- `ux`: related to interface design in general

## A. Acknowledgements

The [RFD template](../templates/prototype.md) is adapted from [HashiCorp's RFC Template](https://www.hashicorp.com/en/how-hashicorp-works/articles/rfc-template).
