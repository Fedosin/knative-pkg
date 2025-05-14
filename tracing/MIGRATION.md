# Migration from OpenCensus `pkg/tracing` to OpenTelemetry `pkg/tracingotel`

## 1. Rationale for Migration

This package has been migrated from OpenCensus to OpenTelemetry because OpenCensus is deprecated and OpenTelemetry is the next major version and future of tracing and metrics collection. OpenTelemetry provides a unified set of APIs and SDKs for observability, and is actively maintained with a broader feature set and community support.

Benefits of migrating to OpenTelemetry include:
- Long-term support and active development.
- A richer ecosystem of instrumentation libraries and exporters.
- Alignment with industry standards for observability.
- Semantic conventions for common operations and attributes.

## 2. Key API Differences

The core way to initialize and use tracing has changed:

### Initialization and Shutdown
- **Old (OpenCensus)**: Involved `tracing.NewOpenCensusTracer()`, `tracer.ApplyConfig()`, and using `config.Config` to define backends and sampling. Dynamic updates were possible via `SetupPublishingWithDynamicConfig`.
- **New (OpenTelemetry)**:
    - A new `tracingotel/otel.go` file provides the core OTel setup.
    - Tracing is initialized by calling `tracingotel.Init(serviceName, cfg *config.Config, logger *zap.SugaredLogger, extraProcessors ...sdktrace.SpanProcessor)`.
        - `serviceName` sets the `service.name` OTel resource attribute.
        - `cfg` (from `knative.dev/pkg/tracingotel/config`) is used to configure sampling (based on `Debug` and `SampleRate` fields) and the Zipkin exporter (if `Backend` is `Zipkin` and `ZipkinEndpoint` is provided).
        - `logger` is used for internal logging, e.g., by the Zipkin exporter.
        - `extraProcessors` allows passing additional OTel `SpanProcessor`s, primarily for testing with in-memory exporters.
    - Tracing is shut down by calling `tracingotel.Shutdown(ctx context.Context)`.
    - The setup functions in `tracingotel/setup.go` (e.g., `SetupPublishingWithStaticConfig`, `SetupPublishingWithDynamicConfig`) have been updated to use this new `tracing.Init`. Dynamic configuration attempts to re-initialize OpenTelemetry, which may have different behavior than the OpenCensus dynamic updates.

### HTTP Server Middleware
- **Old (OpenCensus)**: `tracing.HTTPSpanMiddleware` used `go.opencensus.io/plugin/ochttp.Handler`.
- **New (OpenTelemetry)**: `tracingotel.HTTPSpanMiddleware` (in `tracingotel/http.go`) now uses `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp.NewHandler`.
    - Conditional sampling of paths (via `HTTPSpanIgnoringPaths`) is implemented using `otelhttp.WithFilter`.

### Span Creation (Illustrative - for users of the library directly creating spans)
- **Old (OpenCensus)**: `ctx, span := trace.StartSpan(ctx, "name")`
- **New (OpenTelemetry)**: `ctx, span := tracingotel.Tracer().Start(ctx, "name")`
    - `tracingotel.Tracer()` returns an OTel tracer from the global provider, configured with the instrumentation scope name "knative.dev/pkg/tracing".
    - Span attributes, events, and status setting have new OTel equivalents:
        - `span.Annotate([]trace.Attribute{...}, "msg")` ⟶ `span.AddEvent("msg", trace.WithAttributes(...))`
        - `span.SetStatus(err, msg)` ⟶ `span.SetStatus(codes.Error, msg)` (Note: `codes.Error` comes from `go.opentelemetry.io/otel/codes`)

### Context Propagation
- **Old (OpenCensus)**: Custom propagation logic like `HTTPFormatSequence` and `tracecontextb3` was used, based on `go.opencensus.io/trace/propagation.HTTPFormat`.
- **New (OpenTelemetry)**:
    - The package now relies on the global OpenTelemetry `TextMapPropagator`.
    - It is recommended to configure a global composite propagator (e.g., in `TestMain` for tests, or in `main()` for applications) that includes `propagation.TraceContext{}` (W3C) and `b3.New()` (B3):
      ```go
      import (
          "go.opentelemetry.io/otel"
          "go.opentelemetry.io/otel/propagation"
          "go.opentelemetry.io/contrib/propagators/b3"
      )

      func main() {
          // ...
          prop := propagation.NewCompositeTextMapPropagator(
              propagation.TraceContext{}, // W3C Trace Context
              b3.New(b3.WithInjectEncoding(b3.B3MultipleHeader | b3.B3SingleHeader)), // B3 Multiple and Single header
          )
          otel.SetTextMapPropagator(prop)
          // ...
          // Initialize tracing using tracing.Init or tracing.SetupPublishing...
      }
      ```
    - The `otelhttp` instrumentation used in `tracing.HTTPSpanMiddleware` will automatically use the globally configured propagators for extracting and injecting context.

## 3. Customization Hooks

### Sampling
- Sampling is configured via the `config.Config` struct passed to `tracing.Init` (or the setup functions in `tracing/setup.go`).
    - `Config.Debug = true` results in an `AlwaysSample` sampler.
    - `Config.SampleRate` (float between 0.0 and 1.0) results in a `TraceIDRatioBased` sampler.
    - Otherwise, a `NeverSample` sampler is used.
- All samplers are wrapped with `ParentBased` to respect the sampling decisions of parent spans.

### Exporters
- **Zipkin**: If `config.Config.Backend` is set to `config.Zipkin` and `config.Config.ZipkinEndpoint` is provided, `tracing.Init` will automatically configure and register an OpenTelemetry Zipkin exporter.
- **Other Exporters/Custom Processors**: The `tracing.Init` function accepts `extraProcessors ...sdktrace.SpanProcessor` as its last argument. This allows callers to provide their own configured span processors. This is useful for:
    - Setting up different exporters (e.g., OTLP, Jaeger, Prometheus for exemplar attachment - though metrics aren't in scope for this specific migration).
    - Testing with an in-memory exporter (as seen in `tracing/http_test.go` which uses `tracetest.NewInMemoryExporter()` wrapped in a `SimpleSpanProcessor`).
  Example for a custom OTLP exporter (conceptual):
  ```go
  // import (
  //     ...
  //     "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
  //     sdktrace "go.opentelemetry.io/otel/sdk/trace"
  // )
  //
  // func main() {
  //     ctx := context.Background()
  //     otlpExporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithInsecure())
  //     if err != nil {
  //         log.Fatalf("failed to create OTLP exporter: %v", err)
  //     }
  //     ssp := sdktrace.NewSimpleSpanProcessor(otlpExporter)
  //     defer ssp.Shutdown(ctx)
  //
  //     // ... other setup ...
  //     cfg := &config.Config{ /* ... */ }
  //     logger := ...
  //     if err := tracing.Init("my-service", cfg, logger, ssp); err != nil {
  //         log.Fatalf("failed to init tracing: %v", err)
  //     }
  //     defer tracing.Shutdown(ctx)
  //     // ...
  // }
  ```

## 4. Links to OpenTelemetry Go Documentation

- **OpenTelemetry Go Project**: [https://opentelemetry.io/docs/languages/go/](https://opentelemetry.io/docs/languages/go/)
- **API and SDK**: [https://pkg.go.dev/go.opentelemetry.io/otel](https://pkg.go.dev/go.opentelemetry.io/otel)
- **SDK Configuration**: [https://opentelemetry.io/docs/languages/go/sdk-configuration/](https://opentelemetry.io/docs/languages/go/sdk-configuration/)
- **Exporters**: [https://opentelemetry.io/docs/languages/go/exporters/](https://opentelemetry.io/docs/languages/go/exporters/) (and typically found in `go.opentelemetry.io/otel/exporters/` or `go.opentelemetry.io/contrib/`)
- **Instrumentation Libraries (Contrib)**: [https://github.com/open-telemetry/opentelemetry-go-contrib](https://github.com/open-telemetry/opentelemetry-go-contrib) (includes `otelhttp`, propagators like B3, etc.)
- **Semantic Conventions**: [https://opentelemetry.io/docs/specs/semconv/](https://opentelemetry.io/docs/specs/semconv/) 