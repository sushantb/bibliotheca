# Observability

_Observability_ is the ability to infer the internal state of a system based on external outputs. 

There are 3 pillars of Observability - Traces, Metrics and Logs.

## Traces and Metrics

[OpenTelemetry](https://opentelemetry.io/) (often referred to as OTel) is a vendor-neutral open-source observability framework. It allows developers to instrument, generate, collect, and export signals of different kinds. 

Currently, OpenTelemetry defines four different types of “signals”:

1. **Metrics**: Aggregated (numeric) values over a period of time. (E.g., CPU utilization, incoming HTTP requests per second, …).
2. **Traces**: Collection of linked spans illustrating how requests “flow” through a distributed application.
3. **Logs**: Timestamped (structured or unstructured) messages emitted by services (code) or dependent components.
4. **Baggage**: Contextual information passed between spans.

For traces and metrics, OpenTelemetry takes the approach of a clean-sheet design, specifies a new API, and provides full implementations of this API in multiple language SDKs. 

OpenTelemetry’s approach with logs is different. Because existing logging solutions are widespread in language and operational ecosystems, OpenTelemetry acts as a “bridge” between the logs and other OpenTelemetry components. 

## Logs

The .NET ILogger infrastructure supports fast structured logging, flexible configuration, and a collection of common sinks including the console. Additionally, the ILogger interface can also serve as a facade over many third party logging implementations that offer rich functionality and extensibility.

[Serilog](https://serilog.net/), a third party logging implementation for .NET, provides diagnostic logging to files, the console, and more. It is easy to set up, has a clean API, and is portable between .NET platforms. Serilog supports sematic logging or structured logging.

### Components

| Component         | Description  |
| ------            | --------     |
| Application       | The application being instrumented for observability |
| SDK               | *OpenTelemetry SDK*: The application uses the SDK to exercise APIs for trace and metrics data export. |
|                   | *Serilog SDK*: The application uses the SDK to exercise APIs for log export. Serilog provides several "sinks" (E.g. files, console, OTLP, etc.) to export logs. |
| Collector         | *OpenTelemetry Collector*: It is a vendor-agnostic implementation of how to receive, process and export telemetry data to any observability consumer systems of choice. It removes the need to run, operate, and maintain multiple agents/collectors. |
| Consumer System   | It consists of well-known observability platforms or products. For example, *Jaeger*, or *Zipkin* for traces, *Prometheus* for metrics, *Grafana-Loki* for logs and *Grafana* for visualization. |

## Instrumenting applications

### APIs

.NET provides logging, metrics and distributed tracing APIs in the framework. The .NET OpenTelemetry implementation uses these platforms APIs for instrumentation.
* [Microsoft.Extensions.Logging.ILogger<TCategoryName>](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger-1) for [logging](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging)
* [System.Diagnostics.Metrics.Meter](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.metrics.meter) for [metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/metrics-instrumentation)
* [System.Diagnostics.ActivitySource](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.activitysource) and [System.Diagnostics.Activity](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.activity) for [distributed tracing](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing)

## Configuration

- Refer to [ObservabilityExtensions.cs](../src/Observability/ObservabilityExtensions.cs)

- The following methods are used to add Traces, Metrics and Logs configuration
  - `AddTracing`
  - `AddMetrics`
  - `AddSerilog`
```csharp
    // Add OpenTelemetry Trace and Metrics
    builder.Services
        .AddOpenTelemetry()
        .AddTracing(observabilityOptions)
        .AddMetrics(observabilityOptions);
    
    // Add Serilog for Logs
    builder.Host.AddSerilog();
```

- The `public` method `AddObservability` is used by an application to add all the above configurations in a single method call
```csharp
    var builder = WebApplication.CreateBuilder(args);

    // Add Observability support
    builder.AddObservability();
```
  
- The `public` methods `AddCustomTraces` and `AddCustomMetrics` are used by an application to add custom traces and metrics
```csharp
    builder
        .AddCustomTraces(CustomTraces.Default.Name)
        .AddCustomMetrics(CustomMetrics.Default.Name);
```

### Configure Tracing

The `AddAspNetCoreInstrumentation` and `AddHttpClientInstrumentation` methods add auto instrumentation for ASP.NET Core incoming and outgoing HTTP requests.  `AddJaegerExporter` sends all spans to *Jaeger*. `AddConsoleExporter` helps initial troubleshooting (during development) by writing all traces to STDOUT. `AddOtlpExporter` is used to send all traces using OTLP to the *OpenTelemetry Collector*. The `AlwaysOnSampler` is set to sample all trace spans.

Refer method `AddTracing` in [ObservabilityExtensions.cs](../src/Observability/ObservabilityExtensions.cs).

#### Filtering Traces

Sometimes there is need to filter certain traces. For example, Health endpoints are typically invoked in a very short interval (every 30 seconds, for example) by the surrounding platform to ensure the application is still alive and healthy. Each request would produce traces and flood our observability system. A [CustomProcessor](../src/Observability/Processors/CustomProcessor.cs) implementation prevents such traces from being reported.

### Configure Metrics

Similar to the configuration of Traces, metrics are added in the method `AddMetrics` in [ObservabilityExtensions.cs](../src/Observability/ObservabilityExtensions.cs).

The metrics are more convenient to browse using the `/metrics` endpoint (instead of using the `ConsoleExporter`) for troubleshooting during development. To expose the `/metrics` endpoint, typically scraped by Prometheus based on a configurable interval, the `UseOpenTelemetryPrometheusScrapingEndpoint` middleware is added to `Program.cs` (must be before calling `app.Run()`).

`AddJaegerExporter` sends all spans to *Prometheus* and `AddOtlpExporter` is used to send all metrics using OTLP to the *OpenTelemetry Collector*. 

```csharp
    var app = builder.Build();

    app.UseOpenTelemetryPrometheusScrapingEndpoint();
```

### Configure Logging

The method `AddSerilog` in [ObservabilityExtensions.cs](../src/Observability/ObservabilityExtensions.cs) adds configuration for application logs.

In the configuration *Enrichers* and *Sinks* are added. `WriteTo.Console()` to write logs to the console for troubleshooting during development. `WriteTo.OpenTelemetry()` sends all logs using OTLP to the *OpenTelemetry Collector*. 

## Custom signals with OpenTelemetry

Custom signals may be provided to gain tailored insights about things happening within the application. 

### Custom Traces
Custom trace spans are created in OpenTelemetry by using a an instance of a custom `ActivitySource` to create a new `Activity`. 

### Custom Baggage
A `Baggage` provides and used for the transport of contextual information when producing tracing, metrics, and logs in distributed applications. Technically, `Baggage` is a set of key/value pairs that can, for example, be attached to an `Activity`. The OpenTelemetry SDK provides corresponding `AddBaggage`, `SetBaggage`, and `GetBaggageItem` methods on `Activity`. 

### Custom Metrics
Custom metrics provide important insights about the actual usage of your application or act as additional data points that you can correlate to existing metrics. To collect custom metrics, a `Meter` needs to be created and the metrics to be collected need to be described. 