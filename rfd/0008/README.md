# [RFD] Implementing Telemetry in the ZINC Grading System

---

**Authors**: Thomas Li
**State**: discussion
**Discussion**: [#6](https://github.com/zinc-sig/affairs/pull/6)
**Labels**: `infrastructure`, `metrics`

## Overview

This RFD proposes the implementation of telemetry in the ZINC grading system to enhance monitoring, debugging, and performance analysis across its distributed components. By integrating distributed tracing, centralized logging, and metrics collection, we aim to ensure reliable and efficient grading of student programming assignments, addressing challenges in system scalability and error tracking.

## Background

The ZINC grading system automates the evaluation of programming assignments, operating similarly to a continuous integration pipeline with multiple distributed components processing submissions asynchronously. As the system scales, maintaining reliability and performance becomes increasingly complex. Currently, debugging requires manually inspecting logs from individual components, which is inefficient, particularly for errors spanning multiple services. The absence of centralized monitoring hinders assessing system health or identifying bottlenecks.

Telemetry is essential to address these issues by enabling:

- **Distributed Tracing**: Tracking requests across components to simplify debugging in a distributed environment.
- **Centralized Logging**: Aggregating logs from all workers for easier analysis and correlation.
- **Metrics Collection**: Monitoring performance indicators like grading throughput, error rates, and resource usage for proactive optimization.

Adopting an observability stack, such as Grafana’s LGTM (Loki for logs, Grafana for visualization, Tempo for tracing, and Mimir/Prometheus for metrics), will provide robust tools for visualization, alerting, and historical analysis, enhancing ZINC’s maintainability and performance.

## Proposal

This RFD proposes implementing telemetry in the ZINC grading system to improve observability, debugging, and performance monitoring, with an initial focus on the `core` API monolith (implemented in Go using the Echo framework) and Temporal workflows. By leveraging [OpenTelemetry](https://opentelemetry.io/docs/) for instrumentation and Grafana’s LGTM stack ([Grafana Observability](https://grafana.com/oss/)) for data storage and visualization, we aim to:

- **Trace requests** across the `core` API and Temporal workflows to pinpoint errors in the distributed, asynchronous grading pipeline.
- **Collect metrics** to monitor system performance, such as request rates, error rates, and workflow execution times.
- **Centralize logging** to aggregate logs from all components for easier analysis.
- **Deploy infrastructure** using Grafana’s LGTM stack to support telemetry data storage and visualization.

This approach will enhance ZINC’s reliability, scalability, and maintainability, particularly during peak loads like assignment deadlines.

## Implementation Details

### Tracing

Tracing will be implemented using the [OpenTelemetry Go SDK](https://opentelemetry.io/docs/languages/go/) to track the lifecycle of grading tasks across ZINC’s components.

- **Echo Integration**: Use OpenTelemetry’s [Echo middleware](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/github.com/labstack/echo/oteltrace) to start a trace for each HTTP request to the `core` API. This captures request entry points, such as submission uploads or grading triggers.
- **Temporal Integration**: Leverage [Temporal’s OpenTelemetry integration](https://docs.temporal.io/observability#opentelemetry) to trace workflow and activity executions, capturing key stages like compilation, testing, and grading.
- **Span Guidelines**:
  - Start a new span for calls to external services (e.g., database queries, external APIs) and critical operations (e.g., test execution, grade calculation).
  - The callee function is responsible for starting its span, using a `defer` statement to ensure proper closure, even in error cases. For example:
    ```go
    func processSubmission(ctx context.Context, submissionID string) error {
        ctx, span := tracer.Start(ctx, "processSubmission")
        defer span.End()
        // Process submission logic
        return nil
    }
    ```
  - Use Go’s context to propagate trace information and start subspans for nested operations.
- **Context Propagation**: Ensure trace context is propagated across services, using OpenTelemetry’s propagation mechanisms for HTTP requests and Temporal workflows.

### Metrics

Metrics will be collected using OpenTelemetry and exported to [Prometheus](https://prometheus.io/) or [Grafana Mimir](https://grafana.com/oss/mimir/) for storage and visualization in [Grafana](https://grafana.com/).

- **Proposed Metrics**:

  - **HTTP Requests**: Request count, error count, and latency histograms for `core` API endpoints (e.g., `/submit`, `/grade`).
  - **Temporal Workflows**: Workflow execution count, failure count, and duration for grading tasks.
  - **System Metrics**: CPU and memory usage for the `core` API and Temporal workers, if applicable.
  - **Grading Pipeline**: Queue length for pending submissions, average grading time per submission.

- **Collection**: Use OpenTelemetry’s Go SDK to instrument metrics, such as:

  ```go
  func enqueueSubmission(ctx context.Context, submissionID string) error {
      queueLengthCounter, _ := meter.Int64Counter("queue.length")
      queueLengthCounter.Add(ctx, 1, metric.WithAttributes(attribute.String("submissionID", submissionID)))
      return nil
  }
  ```

- **Monitoring and Alerts**:

  - Create Grafana dashboards to visualize metrics like request rates, error rates, and grading times.
  - Define alert conditions, such as:
    - Error rate > 5% of requests.
    - Average grading time > 10 seconds.
    - Queue length > 100 submissions.

- **Discussion**:
  - Decide on alert channels (e.g., Slack, email).
  - Come up with list of useful metrics to consider

### Logging

Logging will continue using the [Zap logger](https://pkg.go.dev/go.uber.org/zap) with OpenTelemetry integration via [zap-otel](https://pkg.go.dev/go.opentelemetry.io/contrib/bridges/zap).

- **Log Levels**:
  - **Debug**: Detailed information for development (e.g., “Parsing submission JSON”).
  - **Info**: General operational events (e.g., “Submission received: ID=123”).
  - **Warn**: Potential issues not affecting operation (e.g., “Submission retry attempted”).
  - **Error**: Errors impacting operation (e.g., “Failed to compile submission: timeout”).
- **Trace Integration**: Include trace IDs in logs by passing the Go context to log statements:
  ```go
  func logSubmission(ctx context.Context, logger *zap.Logger, submissionID string) {
      logger.Info("Processing submission", zap.String("submissionID", submissionID), zap.Any("context", ctx))
  }
  ```
- **Exporting Logs**: Configure zap-otel to export logs to [Grafana Loki](https://grafana.com/oss/loki/) while maintaining local logging to console and files for compatibility.

## Infrastructure

The telemetry backend will use Grafana’s LGTM stack, deployed via Docker images and managed with [Nomad](https://www.nomadproject.io/), consistent with ZINC’s existing services.

- **Deployment**: Use Nomad to deploy Docker images, ensuring scalability and integration with ZINC’s infrastructure.
- **Data Persistence**: Discuss options for persisting telemetry data, such as:
  - Persistent volumes in Nomad for Loki and Tempo databases.
  - External storage solutions (e.g., object storage).
  - Retention policies (e.g., 7 days, 30 days) to balance storage costs and debugging needs.
