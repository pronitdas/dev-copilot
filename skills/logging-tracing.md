# Logging and Tracing Standards

## Core Requirement

**Always log with trace IDs.**

- Every request has a unique trace ID
- Log context must be propagated across services
- Structured logging only — no unstructured text
- Logs must be machine-parseable

## Implementation

### Structured Logging

```typescript
logger.info('Operation completed', {
  request_id: requestId,
  user_id: userId,
  operation: 'create_marker',
  duration_ms: endTime - startTime,
  status: 'success',
  metadata: {
    marker_id: marker.id,
    // Avoid logging sensitive data
  }
});

logger.error('Operation failed', {
  request_id: requestId,
  error: {
    message: error.message,
    code: error.code,
    stack: error.stack,
  },
});
```

### Trace Propagation

- Use OpenTelemetry for distributed tracing
- Consistent correlation IDs passed through all service calls
- Span context propagated across async boundaries
- Trace sampling for production traffic

### Log Levels

- **ERROR**: Application failures requiring immediate attention
- **WARN**: Degraded service, may need attention
- **INFO**: Key business events, state transitions
- **DEBUG**: Detailed information for debugging
- **TRACE**: Very verbose, development only

## Enforcement

- Log validation in CI pipeline
- Alerting on trace discontinuities
- Trace sampling for all production traffic
- Log retention policies enforced
- Missing trace IDs trigger warnings

## Observability Stack

- **Metrics**: Prometheus metrics on all services
- **Logs**: Structured JSON to stdout/stderr
- **Traces**: OpenTelemetry for request flows
- **Alerts**: SLO-based alerting, not raw metrics

## Key Metrics (The 4 Golden Signals)

- **Latency**: How long to respond?
- **Traffic**: Requests per second?
- **Errors**: Failure percentage?
- **Saturation**: Resource utilization?
