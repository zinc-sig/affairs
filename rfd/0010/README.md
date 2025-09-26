---
authors: Thomas Li
state: prediscussion
discussion:
labels: infrastructure
---

# [RFD] Temporal Workflow Orchestration Integration

## Overview

This Request for Discussion (RFD) proposes the integration of Temporal workflow orchestration into ZINC to enable robust batch processing capabilities and provide workflow management infrastructure for extensions. The integration aims to leverage Temporal's durable execution model to handle complex, long-running workflows such as batch grading of submissions, while maintaining ZINC's modular extension architecture. This proposal builds upon the extension system established in RFD 0009 and provides the workflow orchestration layer necessary for reliable, scalable processing of academic workflows.

## Background

The current ZINC architecture, as enhanced by the extension system (RFD 0009), provides a modular, event-driven platform for academic grading and submission management. However, several limitations exist in handling complex, multi-step workflows:

### Current Limitations

**Lack of Orchestration Capabilities**: While the event-driven architecture enables loose coupling between extensions, there is no built-in mechanism for orchestrating complex, multi-step workflows that may span hours or days. Processing batch submissions requires coordination across multiple services with proper error handling and retry logic.

**State Management Complexity**: Managing the state of long-running processes across distributed extensions is challenging. Without a proper workflow engine, extensions must implement custom state machines, leading to duplicated effort and potential inconsistencies.

**Limited Visibility and Debugging**: When processing fails mid-workflow, understanding the current state and debugging issues is difficult without centralized workflow tracking and history.

**Scalability Constraints**: The current system lacks sophisticated mechanisms for rate limiting, concurrent execution control, and resource management when processing large batches of submissions.

### Why Temporal?

Temporal provides a proven solution for workflow orchestration with the following benefits:

- **Durable Execution**: Workflows maintain state across failures, network partitions, and process restarts
- **Built-in Retry Logic**: Configurable retry policies for activities with exponential backoff
- **Workflow Versioning**: Safe deployment of workflow changes without breaking in-flight executions
- **Observability**: Complete visibility into workflow execution history and current state
- **Language-Native Development**: Workflows are written in Go, maintaining consistency with ZINC's technology stack

The integration of Temporal aligns with ZINC's architectural principles of modularity and separation of concerns, while providing the robustness required for mission-critical academic workflows.

## Proposal

The proposed Temporal integration follows a cross-task-queue architecture where workflows represent high-level orchestration logic at the top level, while activities remain within their respective extensions. This approach leverages Temporal's native capabilities for cross-service communication.

### High-Level Architecture

The integration consists of four key architectural decisions:

1. **Top-Level Workflows**: Workflows are high-level orchestration concerns that live in the top-level `workflow/` package, representing overall use case flows that may span multiple extensions.

2. **Extension-Specific Activities**: Activities remain within their respective extensions, colocated with the domain services they use. Each activity runs on its extension's task queue, ensuring direct access to local resources.

3. **Cross-Task-Queue Execution**: Workflows orchestrate activities across different task queues. When a workflow needs to execute an activity from another extension, it specifies that extension's task queue in the activity options.

4. **Temporal as Communication Layer**: For synchronous cross-extension operations, Temporal replaces NATS. Activities can directly access their local services without additional network hops, while NATS remains for asynchronous event-driven patterns.

### Core Components

**Temporal Infrastructure Package** (`internal/temporal/`): Shared infrastructure providing:
- Client wrapper with connection management
- Worker lifecycle management with FX injection for activities and configuration
- Helper types for workflow and activity registration (`AsActivity`, `AsWorkflow`)
- Configuration structures with task queue management

**Top-Level Workflows** (`workflow/`): High-level orchestration logic:
- Batch grading workflows that span submission and pipeline extensions
- Future: Other cross-cutting workflows like exam scheduling

**Extension Activities**: Each extension defines activities colocated with their services:
- Submission extension: `GetCollectionLatestDeliveries` activity
- Pipeline extension: `ExecutePipeline` activity

**Configuration Service Enhancement**: The existing extension configuration service provides Temporal connection details and task queue names to extensions.

## Architecture

### Package Structure

The architecture separates high-level workflows from extension-specific activities:

```
zinc-core/
├── workflow/                  # Top-level orchestration logic
│   ├── grading/
│   │   └── batch_grading.go  # Batch grading workflow
│   └── workflow.go            # FX module for workflow registration
│
├── internal/temporal/         # Shared Temporal infrastructure
│   ├── client.go              # Client wrapper with connection management
│   ├── worker.go              # Worker with FX activity/config injection
│   ├── types.go               # AsActivity/AsWorkflow helper functions
│   └── temporal.go            # FX modules for Temporal setup
│
├── internal/api/submission/   # Submission extension
│   ├── activities/
│   │   └── delivery.go        # GetCollectionLatestDeliveries activity
│   └── submission.go          # FX registration with AsActivity
│
├── internal/api/pipeline/     # Pipeline extension
│   ├── activities/
│   │   └── execution.go       # ExecutePipeline activity
│   └── pipeline.go            # FX registration with AsActivity
│
└── internal/extension/        # Extension configuration service
    └── service.go             # Enhanced with Temporal config and task queues
```

### System Architecture Diagram

```mermaid
graph TB
    subgraph "ZINC Core"
        Core[Core Services]
        ExtConfig[Extension Config Service]
        TopWF[Top-Level Workflows]
        TempInfra[Temporal Infrastructure]
    end

    subgraph "Temporal Server"
        TemporalServer[Temporal Server]
        TemporalDB[(History & Visibility)]
        TQSub[Task Queue: zinc-submission]
        TQPipe[Task Queue: zinc-pipeline]
    end

    subgraph "Submission Extension"
        SubWorker[Worker<br/>zinc-submission]
        SubAct[GetCollectionLatestDeliveries<br/>Activity]
        SubSvc[Submission Service]
        SubDB[(Submission DB)]
    end

    subgraph "Pipeline Extension"
        PipeWorker[Worker<br/>zinc-pipeline]
        PipeAct[ExecutePipeline<br/>Activity]
        PipeSvc[Pipeline Service]
    end

    subgraph "Storage"
        ObjectStore[(Object Storage)]
    end

    subgraph "Message Broker"
        NATS[NATS Server<br/>for async events]
    end

    %% Configuration flow
    ExtConfig -.->|Task queue config| SubWorker
    ExtConfig -.->|Task queue config| PipeWorker

    %% Worker registration
    SubWorker -->|Poll| TQSub
    PipeWorker -->|Poll| TQPipe
    TempInfra -->|Start workflow| TemporalServer

    %% Workflow execution across task queues
    TopWF -->|Execute on TQ| TQSub
    TopWF -->|Execute on TQ| TQPipe

    %% Activity dependencies
    SubAct --> SubSvc
    SubSvc --> SubDB
    PipeAct --> PipeSvc
    PipeAct --> ObjectStore

    %% Async events still use NATS
    Core --> NATS
    SubSvc -.->|Async events| NATS
    PipeSvc -.->|Async events| NATS
```

### Extension Integration Pattern

Extensions interact with Temporal through FX dependency injection:

1. **Configuration Injection**: Extensions receive Temporal configuration and task queue names via FX
2. **Activity Registration**: Activities are registered using `temporal.AsActivity()` in FX modules
3. **Worker Creation**: The shared `internal/temporal/worker.go` creates workers with injected activities and configuration
4. **Task Queue Assignment**: Each extension's worker polls its specific task queue for activities

### Cross-Extension Communication

Temporal enables direct cross-extension communication through task queue routing, eliminating NATS for synchronous operations:

```mermaid
sequenceDiagram
    participant User as User/API
    participant Core as Core/Workflow Starter
    participant BW as Batch Grading Workflow
    participant TSub as Temporal<br/>zinc-submission TQ
    participant TPipe as Temporal<br/>zinc-pipeline TQ
    participant SubAct as GetDeliveries Activity<br/>(Submission Extension)
    participant PipeAct as ExecutePipeline Activity<br/>(Pipeline Extension)
    participant SubSvc as Submission Service
    participant OS as Object Storage

    User->>Core: Request batch grading
    Core->>BW: StartWorkflow(collection_id, config_id)

    Note over BW: Set TaskQueue: zinc-submission
    BW->>TSub: ExecuteActivity(GetCollectionLatestDeliveries)
    TSub->>SubAct: Route to submission worker
    SubAct->>SubSvc: Query latest deliveries by user
    SubSvc-->>SubAct: Delivery list
    SubAct-->>BW: List of delivery IDs

    loop For each delivery
        Note over BW: Set TaskQueue: zinc-pipeline
        BW->>TPipe: ExecuteActivity(ExecutePipeline, delivery_id)
        TPipe->>PipeAct: Route to pipeline worker
        PipeAct->>OS: Pull submission data
        OS-->>PipeAct: Submission content
        PipeAct->>PipeAct: Execute grading pipeline
        PipeAct->>OS: Store grading results
        PipeAct-->>BW: Pipeline execution complete
    end

    BW-->>Core: All deliveries graded
    Core-->>User: Batch results
```

## Configuration Management

The system employs a simple, extensible configuration approach:

### Core Configuration

```yaml
temporal:
  host_port: temporal.example.com:7233  # Combined host:port format
  namespace: zinc-dev                    # Environment-specific
  task_queue_prefix: zinc                # Prefix for all task queues
```

### Extension Configuration Distribution

The extension service provides Temporal configuration through the existing configuration mechanism. Extensions receive the task queue prefix and compute their specific queues:

```protobuf
message Configuration {
  string name = 1;
  map<string, string> properties = 2;
}

// Extension receives:
// name: "temporal"
// properties: {
//   "host_port": "temporal.example.com:7233",
//   "namespace": "zinc-dev",
//   "task_queue_prefix": "zinc"  # Prefix only, not computed queue
// }
```

### Environment Strategy

Namespaces are environment-based to provide clean separation:
- `zinc-dev` for development
- `zinc-staging` for staging
- `zinc-prod` for production

Task queues follow a three-part naming pattern: `{prefix}-{component}-{queue_type}`:
- Core services: `zinc-core-default` (for core workflows)
- Submission extension: `zinc-submission-default` (for submission activities)
- Pipeline extension: `zinc-pipeline-default` (for pipeline activities)

## Task Queue Management

The system uses a consistent three-part naming pattern for task queues: `{prefix}-{component}-{queue_type}`. Each extension defines its task queue suffix as a constant (e.g., `pipeline-default`) and the Temporal client combines it with the environment-specific prefix (e.g., `zinc`) to create the full queue name (e.g., `zinc-pipeline-default`).

Extensions provide their computed task queue as a named FX dependency that the worker consumes, ensuring each extension's activities run on the correct task queue. This approach provides type safety through constants while maintaining flexibility across environments.

## Workflow Patterns

### Batch Processing Pattern

Batch workflows orchestrate activities across different task queues:

```
BatchGradingWorkflow (runs on initiating task queue)
├── Validate batch parameters (collection_id, config_id)
├── Execute GetCollectionLatestDeliveries (on zinc-submission queue)
├── Prepare batch metadata
├── For each delivery (parallel with limits)
│   └── Execute ExecutePipeline (on zinc-pipeline queue)
│       ├── Pull submission from object storage
│       ├── Execute grading pipeline
│       ├── Handle failures with retry
│       └── Store results to object storage
├── Aggregate results
└── Generate batch summary
```

Workflows specify the target task queue when calling activities across extensions:
```go
// Call submission extension activity
ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
    TaskQueue: "zinc-submission-default",
    StartToCloseTimeout: 30 * time.Second,
})
workflow.ExecuteActivity(ctx, "GetCollectionLatestDeliveries", collectionID)

// Call pipeline extension activity
ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
    TaskQueue: "zinc-pipeline-default",
    StartToCloseTimeout: 5 * time.Minute,
})
workflow.ExecuteActivity(ctx, "ExecutePipeline", deliveryID)
```

Key considerations:
- Use of `workflow.Go()` for parallel execution
- Selector pattern for result streaming
- Continue-as-new for very large batches
- Proper error aggregation and reporting

### Activity Design Pattern

Activities are extension-specific and receive dependencies through FX injection. Each extension:
1. Defines task queue constants in `constants.go`
2. Implements activities with injected services in `activities/` directory
3. Registers activities and provides task queue in the module file

Example structure for pipeline extension:
```go
// internal/api/pipeline/activities/execution.go
type ExecutionActivity struct {
    service  pipeline.Service   // Injected via FX
    storage  storage.Client     // Injected via FX
}

func (a *ExecutionActivity) ExecutePipeline(ctx context.Context, deliveryID string) (*Result, error) {
    // Direct access to local services, no NATS needed
}

// internal/api/pipeline/pipeline.go (module file)
var Module = fx.Module(
    "pipeline",
    fx.Provide(
        // ... existing providers
        temporal.AsActivity(NewExecutionActivity),  // Register activity
        fx.Annotate(                                // Provide task queue
            func(client *temporal.Client) string {
                return client.FormatTaskQueue(DefaultTaskQueue)
            },
            fx.ResultTags(`name:"temporalTaskQueue"`),
        ),
    ),
)
```

### Workflow Versioning

The system uses Temporal's native versioning API for handling workflow updates:

```
version := workflow.GetVersion(ctx, "AddRetryLogic", workflow.DefaultVersion, 1)
if version == 1 {
    // New logic with retry
} else {
    // Original logic
}
```

This approach ensures safe updates without breaking in-flight workflows.

## Extension Capabilities

### Full Workflow Ownership

Extensions have complete control over their workflow definitions:

1. **Workflow Definition**: Extensions define workflows specific to their domain
2. **Activity Implementation**: Activities have access to extension-specific dependencies
3. **Worker Configuration**: Extensions control worker options and concurrency
4. **Task Queue Management**: Each extension uses its own task queue

### Workflow Boundaries

Workflows and activities have clear separation:
- Workflows are top-level orchestration logic, not tied to extensions
- Activities are extension-specific and run on their extension's task queue
- Cross-extension interaction happens through Temporal's task queue routing
- No direct service-to-service calls needed for synchronous operations

### Extension Lifecycle

Extensions integrate with Temporal through FX:

```
Extension Startup:
1. Register activities using temporal.AsActivity() in FX module
2. Temporal config and task queue injected via FX
3. Worker created by internal/temporal/worker.go with injected activities
4. Worker starts polling extension's task queue
5. Activities have access to extension's services

Extension Shutdown:
1. FX lifecycle stops worker gracefully
2. Wait for current activity executions
3. Close Temporal client connection
```

## Security Considerations

### Authentication and Authorization

- **mTLS**: Temporal cluster connections use mutual TLS for authentication
- **Namespace Isolation**: Each environment has its own namespace with access controls
- **Activity-Level Authorization**: Activities verify permissions before executing operations

### Data Protection

- **Encryption at Rest**: Temporal database encrypted at rest
- **Sensitive Data Handling**: Avoid storing sensitive data in workflow history
- **Audit Logging**: All workflow executions are logged for audit purposes

### Extension Isolation

- **No Direct Access**: Extensions cannot access other extensions' workflows
- **Configuration Scoping**: Each extension receives only its configuration
- **Task Queue Isolation**: Separate task queues prevent cross-extension interference

## Monitoring and Observability

### Metrics

The integration provides comprehensive metrics:

- **Workflow Metrics**: Start rate, completion rate, failure rate, duration
- **Activity Metrics**: Execution time, retry count, failure reasons
- **Worker Metrics**: Task queue depth, polling success rate, concurrent executions
- **Business Metrics**: Submissions processed, grading throughput, batch sizes

### Tracing

OpenTelemetry integration provides distributed tracing:

```
Trace: Batch Grading Request
├── Span: Start Batch Workflow
├── Span: Validate Batch
├── Span: Process Submission Group
│   ├── Span: Grade Submission Activity
│   ├── Span: NATS Publish
│   └── Span: Store Results
└── Span: Generate Report
```

### Workflow Visibility

Temporal provides built-in visibility features:

- **Workflow Status**: Current state of all workflows
- **Search Capabilities**: Query workflows by various attributes
- **History Replay**: Complete execution history for debugging
- **UI Access**: Temporal Web UI for operational visibility

## Migration Strategy

### Phased Approach

The integration follows a phased migration strategy:

**Phase 1: Core Infrastructure**
- Deploy Temporal infrastructure
- Implement shared Temporal package
- Set up monitoring and observability

**Phase 2: Core Workflow Infrastructure**
- Implement top-level workflow package structure
- Create batch grading workflow in workflow/grading/
- Set up workflow registration with FX

**Phase 3: Extension Activity Implementation**
- Implement GetCollectionLatestDeliveries in submission extension
- Implement ExecutePipeline in pipeline extension
- Register activities using temporal.AsActivity()
- Configure task queues for each extension

**Phase 4: Advanced Features**
- Implement complex workflow patterns
- Add workflow scheduling
- Enhance monitoring and alerting

### Backward Compatibility

The migration maintains backward compatibility:

- Existing NATS events continue to function
- Extensions can adopt Temporal incrementally
- Core APIs remain unchanged
- Gradual migration of batch operations

## Best Practices

### Task Queue Design
- **One task queue per extension** for proper worker-to-activity alignment
- **Colocate activities with services** they depend on for direct access
- **Explicitly route activities** by specifying task queues in workflow activity options
- **Configure appropriate timeouts and retry policies** based on activity characteristics

### Activity Design Principles
- **Direct service access**: Activities use injected local services, avoiding network calls
- **Idempotency**: Design activities to handle retries safely
- **Error handling**: Return non-retryable errors for business logic failures
- **FX integration**: Register activities using `temporal.AsActivity()` in module definitions

## Discussion

Several aspects require further consideration during implementation:

### Resource Management

**Question**: How should we manage resource limits across extensions?

While starting without limits provides flexibility, production deployments will need:
- Worker pool sizing strategies
- Rate limiting for workflow starts
- Resource quotas per extension
- Backpressure mechanisms

### Workflow Granularity

**Question**: What is the appropriate granularity for workflows?

Considerations include:
- Single submission vs. batch workflows
- Activity size and duration
- State management complexity
- Reusability across domains

### Error Handling Strategy

**Question**: How should different error types be handled?

Error categories requiring different strategies:
- Transient errors (network, temporary unavailability)
- Business logic errors (validation failures)
- System errors (out of memory, disk full)
- External service errors (pipeline failures)

### Operational Procedures

**Question**: What operational procedures are needed?

Key operational concerns:
- Workflow termination policies
- Stuck workflow detection
- Manual intervention procedures
- Disaster recovery plans

### Performance Optimization

**Question**: How do we optimize for performance?

Performance considerations:
- Workflow caching strategies
- Activity batching
- Parallel execution limits
- History size management

## Alternatives Considered

### Alternative Workflow Engines

**Apache Airflow**: Popular for data pipelines but:
- Python-based, inconsistent with Go stack
- More suited for scheduled batch jobs
- Less support for long-running workflows

**Cadence**: Temporal's predecessor but:
- Smaller community
- Less active development
- Migration path to Temporal exists

**Custom Solution**: Build workflow engine in-house but:
- Significant development effort
- Maintenance burden
- Reinventing proven solutions

### Alternative Architectures

**Centralized Workflow Service**: Single service managing all workflows but:
- Creates bottleneck
- Violates extension independence
- Increases coupling

**Workflow-per-Submission**: Individual workflows for each submission but:
- Excessive resource usage
- Difficult batch management
- Complex aggregation

## Conclusion

The integration of Temporal workflow orchestration into ZINC addresses critical gaps in handling complex, long-running academic workflows while maintaining the modular architecture established by the extension system. The cross-task-queue architecture ensures clear separation between orchestration logic and domain implementation.

Key benefits of this integration include:

- **Reliability**: Durable execution ensures workflows complete despite failures
- **Scalability**: Controlled concurrency and resource management for large-scale processing
- **Visibility**: Complete observability into workflow execution and state
- **Direct Communication**: Temporal replaces NATS for synchronous cross-extension calls
- **Service Locality**: Activities have direct access to their extension's services
- **Simplified Architecture**: No request/response patterns needed for activity communication
- **Maintainability**: Clear separation between workflows (orchestration) and activities (implementation)

The proposed architecture provides a solid foundation for current requirements while remaining extensible for future needs. The phased migration approach ensures smooth adoption with minimal disruption to existing operations.

## References

1. [Temporal Documentation](https://docs.temporal.io/)
2. [Temporal Go SDK](https://github.com/temporalio/sdk-go)
3. [Workflow Patterns](https://temporal.io/docs/concepts/workflows)
4. [Activity Best Practices](https://temporal.io/docs/concepts/activities)
5. [ZINC RFD 0009: Extension System](../0009/README.md)
6. [Uber FX Dependency Injection](https://uber-go.github.io/fx/)
7. [NATS Messaging](https://nats.io/)
8. [OpenTelemetry Specification](https://opentelemetry.io/docs/)
