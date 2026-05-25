+++
title = "OpenTelemetry Collector configuration for Prometheus"
tags = [OpenTelemetry]
+++

OpenTelemetry can export metrics so as to be scraped by Prometheus. Here is a manifest for an OpenTelemetryCollector object configured to do so:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
spec:
  image: otel/opentelemetry-collector-contrib:0.152.0
  mode: deployment

  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch: {}

    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
      debug:
        verbosity: detailed

    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus, debug]

        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [debug]

        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [debug]
```
