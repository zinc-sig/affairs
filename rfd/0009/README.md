# [RFD] ZINC Extension System

---

authors: Thomas Li
state: discussion
discussion: [#7](https://github.com/zinc-sig/affairs/pull/7)
labels: `infrastructure`

---

## Overview

This Request for Discussion (RFD) proposes the implementation of a modular extension system for ZINC, a system designed to manage grading of programming assignments, collection of student submissions, computer-based examinations, and student scores in university courses. The goal is to transition ZINC from a monolithic architecture to a pluggable, event-driven system where functionalities are encapsulated in independent extensions. This approach aims to enhance flexibility, maintainability, and scalability by allowing components to be developed, updated, and deployed independently. The extension system will leverage an event-driven architecture, with communication facilitated through a message broker, ensuring loose coupling between components.

## Background

The current ZINC system operates as a monolithic application, where functionalities are tightly integrated. This design has several limitations:

- **Limited Flexibility**: Adding new features or modifying existing ones requires changes to the entire codebase, increasing the risk of introducing errors and complicating maintenance.
- **Scalability Challenges**: Scaling a monolithic system is complex, as all components must scale together, even if only one functionality requires additional resources.
- **Maintenance Overhead**: Debugging and updating a tightly coupled system is time-consuming, as changes in one area can have unintended effects on others.

These issues have been highlighted in prior discussions within the ZINC Special Interest Group (SIG), where stakeholders expressed the need for a more modular architecture to support evolving requirements, such as new scoring strategies or additional course-related functionalities. The proposed extension system draws inspiration from modern software design principles, such as modularity and the Open-Closed Principle, which advocate for systems that are open for extension but closed for modification.

By adopting a modular, event-driven architecture, ZINC can address these shortcomings, enabling easier feature development, improved scalability, and better resilience through asynchronous communication. This RFD builds on concepts from prior RFDs, such as the architecture for grader workers (RFD 0004), which proposed splitting the grader into control plane and worker components to improve isolation and resource management.

## Proposal

The proposed extension system for ZINC consists of the following key components:

1. **Core Component**: Manages student-course relationships and abstracts course events as "activities" (e.g., assignments, exams). Each activity can be associated with specific extensions based on its requirements.
2. **Extensions**: Independent modules encapsulating specific functionalities, including:
   - **Grader**: Handles grading of programming assignments.
   - **Submission-Collector**: Manages the collection of student submissions.
   - **Examination**: Controls and manages computer-based examinations.
   - **Potential Additional Extensions**:
     - **Scoring**: Calculates scores based on multiple components, such as test cases or partial credit.
     - **Attendance**: Tracks student attendance for course events.
3. **Event-Driven Communication**: Extensions and the core communicate via events through a message broker, ensuring loose coupling and enabling asynchronous operations.
4. **Message Broker**: A system like NATS, which supports event-based, topic-based subscriptions with at-least-once delivery, will facilitate communication [NATS Documentation](https://nats.io/).

This design aligns with best practices for extensible systems, emphasizing modularity and well-defined interfaces to allow future growth without major architectural changes.

### High-Level Workflow

The system operates as follows:

- **Extension Registration**: Extensions register with the core via an RPC call, providing metadata about the events they emit and subscribe to. The core provides configuration settings, such as message broker connection details.
- **Event Flow**: Extensions react to events published on the message broker. For example:
  - A student submits an assignment via the `submission-collector` extension, which publishes a `submission.received` event.
  - The `grader` extension, subscribed to `submission.received`, initiates the grading process and publishes a `grading.completed` event upon completion.
  - The core, subscribed to `grading.completed`, updates the student's record.
- **State Management**: State changes are triggered by events, not direct RPC calls, to maintain decoupling and ensure consistency.

## Implementation

### Extension Registration

Extensions register with the core using an RPC call, providing metadata such as their identifier, supported events, and event subscriptions. The core responds with configuration data, enabling extensions to integrate with the system. The registration process includes:

- **Metadata Exchange**: Extensions specify the events they emit (e.g., `submission.received`) and subscribe to (e.g., `grading.completed`).
- **Configuration Retrieval**: Extensions obtain configurations, such as message broker settings or dependencies like the Temporal workflow engine for the `grader` extension.
- **Subscription Setup**: Extensions subscribe to relevant event topics on the message broker.

The following is an example of the protocol buffer (protobuf) messages for registration:

```protobuf
syntax = "proto3";
package extension;
option go_package = "../extension";

message RegistrationRequest {
  string id = 1;
}

message RegistrationResponse {
  string status = 1;
}

message ConfigurationRequest {
  repeated string names = 1;
}

message Configuration {
  string name = 1;
  map<string, string> properties = 2;
}

message ConfigurationResponse {
  repeated Configuration configurations = 1;
}

service Extension {
  rpc Register(RegistrationRequest) returns (RegistrationResponse);
  rpc GetConfiguration(ConfigurationRequest) returns (ConfigurationResponse);
}
```

### Event System

The event system is the backbone of communication, using a message broker that supports:

- **Event-Based Subscriptions**: Clients subscribe to specific events.
- **Topic-Based Filtering**: Events are organized by topics (e.g., `submission.*` for submission-related events).
- **At-Least-Once Delivery**: Ensures reliable message delivery with redelivery mechanisms for failures.

[NATS](https://nats.io/) is a candidate for the message broker due to its lightweight design, mature Go client, and support for publish-subscribe patterns]. The event flow ensures that extensions operate independently, reacting only to relevant events without direct dependencies.

### Configuration Management

Configurations are managed centrally by the core, allowing extensions to request settings during registration or runtime. This centralization supports dynamic updates, enabling extensions to adapt to changes without restarting. For example, the `grader` extension might request Temporal workflow configurations, while the `submission-collector` might need storage settings.

### Event Hierarchy

Events are categorized by scope to facilitate organization and filtering:

- **Activity-Scoped Events**: Related to specific tasks, such as grading a single assignment (e.g., `grading.completed`).
- **Offering-Scoped Events**: Related to broader course activities, such as aggregating scores for a course (e.g., `course.score.updated`).

This hierarchy ensures that extensions can subscribe to relevant events efficiently, reducing unnecessary processing.

### Example Event Flow

| Step | Component            | Action                      | Event Published       |
| ---- | -------------------- | --------------------------- | --------------------- |
| 1    | Submission-Collector | Receives student assignment | `submission.received` |
| 2    | Grader               | Initiates grading process   | -                     |
| 3    | Grader               | Completes grading           | `grading.completed`   |
| 4    | Core                 | Updates student record      | -                     |

This table illustrates the decoupled nature of the system, where each component reacts to events without direct interaction.

## Discussion

Several aspects of the proposed design require further exploration:

1. **Component Scope Definition**: Clearly defining the responsibilities of the core and each extension to prevent overlap and ensure all functionalities are covered. For example, should the `scoring` extension handle all score calculations, or should some logic reside in the core?
2. **Configuration Negotiation**: Developing a mechanism for dynamic configuration updates and dependency injection. How should extensions handle changes to configurations, such as updated message broker settings?
3. **Event Hierarchy and Naming**: Establishing standardized naming conventions and event structures to simplify subscription and management. Should events follow a specific format, such as `namespace.action`?
4. **Error Handling and Reliability**: Ensuring the event system handles failures gracefully, with mechanisms for retries, error reporting, and recovery. How should the system handle a failed grading process?

These topics will be critical during the discussion phase to refine the design and ensure a robust implementation.

## Conclusion

The proposed extension system for ZINC represents a significant step toward a more modular, scalable, and maintainable architecture. By adopting an event-driven approach with pluggable extensions, ZINC can better support evolving requirements, such as new grading strategies or additional course functionalities. The use of a message broker like NATS ensures reliable, asynchronous communication, while centralized configuration management simplifies extension integration. Further discussion is needed to address open questions and finalize the implementation plan, but this design aligns with best practices for extensible systems, promising improved flexibility and resilience for ZINC.
