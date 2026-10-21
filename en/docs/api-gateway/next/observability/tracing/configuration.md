---
title: "Tracing Configuration Options"
description: "Configure distributed tracing for API Platform Gateway: sampling rates, OTLP collector endpoints, and batch export settings."
canonical_url: https://wso2.com/api-platform/docs/api-gateway/next/observability/tracing/configuration/
md_url: https://wso2.com/api-platform/docs/api-gateway/next/observability/tracing/configuration.md
tags:
  - api-gateway
  - observability
  - tracing
author: WSO2 API Platform Documentation Team
last_updated: 2026-10-20
content_type: "reference"
---

# Tracing Configuration

## Configuration Options

### Export Protocol

The router and the policy engine export spans with OTLP over gRPC by default. To export to a collector that accepts only OTLP over HTTP, set `protocol` to `http` and point `endpoint` at the collector's HTTP port:

```toml
[tracing]
enabled = true
protocol = "http"                  # "grpc" (default) or "http"
endpoint = "otel-collector:4318"   # 4317 for grpc, 4318 for http
```

With `http`, spans are sent as OTLP protobuf to `/v1/traces` on the endpoint. If the endpoint has no port, the standard port for the protocol is used.

### Adjusting Sampling Rate

To reduce trace volume in high-traffic environments, adjust the sampling rate:

```toml
[tracing]
sampling_rate = 0.1  # Sample 10% of requests
```

Sampling strategies:
- `1.0` (100%): Sample all requests - recommended for development and low-traffic environments
- `0.5` (50%): Sample half of requests - moderate traffic
- `0.1` (10%): Sample 10% of requests - high traffic
- `0.01` (1%): Sample 1% of requests - very high traffic

**Note:** Lower sampling rates reduce overhead but may miss important traces.

The router makes the sampling decision. The policy engine follows it, so a request's spans are either all kept or all dropped. The rate applies only to a request that arrives with no trace context. A request whose `traceparent` header carries a sampling decision keeps that decision.


### Custom Service Names

Customize the service names the router and the policy engine report:

```toml
[router]
tracing_service_name = "router-prod-us-east-1"

[policy_engine]
tracing_service_name = "policy-engine-prod-us-east-1"
```

### Batch Configuration

Optimize batch settings for your environment:

```toml
[tracing]
batch_timeout = "5s"            # Wait up to 5s before exporting
max_export_batch_size = 1024    # Export up to 1024 spans per batch
```

**Lower timeout**: Faster trace visibility, more network overhead
**Higher timeout**: Better batching efficiency, slower trace visibility

## Customizing OpenTelemetry Collector

The OTLP Collector configuration is located at:
```
gateway/observability/otel-collector/config.yaml
```

### Configuration Structure

The configuration consists of three main sections:

#### Receivers
Define how traces are received:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

#### Processors
Transform and enrich traces:

```yaml
processors:
  # Batch spans for efficiency
  batch:
    timeout: 1s
    send_batch_size: 1024

  # Add resource attributes
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
      - key: cluster
        value: us-west-2
        action: upsert

  # Memory limiter to prevent OOM
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

  # Sampling processor
  probabilistic_sampler:
    sampling_percentage: 10  # Sample 10% of traces
```

#### Exporters
Define where traces are sent:

```yaml
exporters:
  # Send to Jaeger
  otlp:
    endpoint: jaeger:4317
    tls:
      insecure: true

  # Debug output to console
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200
```

#### Service Pipeline
Connect receivers, processors, and exporters:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [otlp, debug]
```

### Example: Multi-Backend Export

Send traces to multiple backends simultaneously:

```yaml
exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  datadog:
    api:
      key: ${DD_API_KEY}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/jaeger, otlp/tempo, datadog]
```

### Example: Tail-Based Sampling

Keep all error traces but sample successful traces:

```yaml
processors:
  tail_sampling:
    policies:
      - name: error-traces
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000
      - name: probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

## Trace Context Propagation

The gateway components automatically propagate trace context using standard W3C Trace Context headers:

- `traceparent`: Contains trace ID, span ID, and sampling decision
- `tracestate`: Contains vendor-specific trace information

When making requests to the gateway, you can:

1. **Let the gateway create a new trace** (default)
2. **Propagate your own trace context** by including trace headers:

```bash
curl http://localhost:8080/weather/v1.0/us/seattle \
  -H "traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01"
```

This allows you to trace requests across your entire system, including services before and after the gateway.

Each request is one trace. The policy engine's spans are children of the router's span for the request, so you see the router, every policy that ran, and the upstream call in one tree. A streamed request or response body is one span for the whole stream, with one span per policy. Policies that call other services through the gateway's shared HTTP client get a client span for each call, and the call carries the trace context to that service.

Analytics events carry the trace too. A sampled request's analytics event includes its trace ID and span ID, so you can go from an event to its trace. The OpenTelemetry analytics publisher sets them as the log record's `traceId` and `spanId`, and traffic logs include `traceId`.

