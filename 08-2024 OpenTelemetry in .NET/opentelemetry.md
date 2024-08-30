# OpenTelemetry in .NET

The OpenTelemetry website defines the OpenTelemetry as:
> OpenTelemetry is a collection of APIs, SDKs, and tools. Use it to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) to help you analyze your software’s performance and behavior.

That's actually quite a precise summary, but maybe a bit technical, so let me rephrase what the OpenTelemetry is in my words. The OpenTelemetry is an abstraction for metrics, logs, and traces. With the OpenTelemetry we are abstracted from a need to install any vendor-specific agent or SDK that would limit us in what it is capable of collecting. As we are abstracted on a vendor-specific collector, we are also abstracted from observability systems like Datadog, Elastic Observability, and Azure Monitor/Application Insights. In other words, thanks to OpenTelemetry we are vendor-free and we can switch between vendors based on current limitations, price, or simply our needs and preferences.

---

The OpenTelemetry is made by a language-specific SDK that collects application metrics like logs, traces and metrics. The SDK sends these data to a collector using the OTLP protocol. The collector may be a vendor-specific collector that is part of an observability solution like Elastic APM, or the collector may be the OpenTelemetry Collector. The OpenTelemetry Collector is our main area of interest.

![The OpenTelemetry diagram](./assets/otel-diagram.svg "The OpenTelemetry diagram")

The OpenTelemetry Collector is a running daemon that is highly configurable and made of three main parts.

1. Receivers
2. Exporters
3. Processors

## Receivers
The Receiver is a component that can collect or receive signals. OpenTelemetry offers a long list of receivers containing receivers that can collect host metrics, NGINX logs, RabbitMQ logs, Datadog Agent and among other things also the OTLP.

## Exporters
The Exporter is in direct opposition to the receiver. The Exporter is responsible for exporting signals down in the pipeline. The target may be a storage like ElasticSearch or a Sender that pipes signals down in the line. In the following examples, we use an OTLP exporter that forwards signals to yet another OTLP collector.

## Processors
The Processor takes data from receivers and transforms, filters, drops, renames or even recalculates signals before sending them to the exporters. Typical and the most often used processors are `batch` and `memory_limiter`.

## Run OpenTelemetry Collector
The sample used here is made by an application that is the telemetry producer with installed OpenTelemetry SDK to send the telemetry signals to the OpenTelemetry Collector which forwards them to the Aspire for data visualization.

The configuration for such a sample follows.

```yaml
receivers:
  # This is a configuration for OTLP receiver.
  otlp:
    protocols:
      grpc:
        # In the newer version the receiver is out of the box restricted to listen to the localhost.
        # Our receiver will run in the Docker so we must allow any connection.
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  # The debug exporter is handy for troubleshooting.
  debug:
    verbosity: detailed

  # The OTLP exporter sends signals to the Aspire.
  otlp:
    endpoint: "aspire:18889"
    tls:
      insecure: true

# In our example we don't use any processors, in a real scenariou you would at least use the `batch` and `memory_limit` processors.
service:
  pipelines:
    # Traces are handle in this pipeline. 
    # Every pipeline can have N receivers. In our example I have just OTLP receiver, but depending on your needs
    # you could have for example the PostgreSQL receiver that collects metrices from the PostgreSQL databases.
    metrics:
      receivers: [otlp]
      exporters: [otlp]
    traces:
      receivers: [otlp]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      # Here we have an example of multiple exporters
      exporters: [debug, otlp]

```

I run the OpenTelemetry Collector in the Docker. To see the data I use for local development the Aspire. Let's prepare a docker-compose file with OpenTelemetry Collector and Aspire.

```yaml
version: '3.8'

services:
  otel-collector:
    container_name: otel-collector
    image: otel/opentelemetry-collector-contrib:latest
    command: [ "--config", "/etc/otel/otel-configuration.yaml" ]
    ports:
      - "4317:4317" # For gRPC
    volumes:
      - ./:/etc/otel
  
  aspire:
    container_name: aspire
    image: mcr.microsoft.com/dotnet/aspire-dashboard:8.1.0
    environment:
      - DOTNET_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS=true
    ports:
      - "18888:18888" # For UI
      - "18889:18889" # For OTLP protocol
    depends_on:
      - otel-collector
```

### .NET ecosystem
First of all install `OpenTelemetry.Exporter.OpenTelemetryProtocol`, `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.Http`, `OpenTelemetry.Instrumentation.Runtime` NuGet packages. Please notice how the packages are named. Simply based on name you already know what the package offers. Every OpenTelemetry SDK supports **code-based** or **zero-code** configuration. In the sample, I use the code-based configuration.

In the `program.cs` add the OpenTelemetry setup.

```csharp
var configuration = builder.Configuration;
var serviceName = configuration.GetValue("ServiceName", defaultValue: "Sample")!;
var serviceVersion = typeof(Program).Assembly.GetName().Version?.ToString() ?? "dev";
var serviceInstanceId = Environment.MachineName;
var local = (OtlpExporterOptions options) =>
{
    options.Protocol = OtlpExportProtocol.Grpc;
    options.Endpoint = new Uri("http://localhost:4317");
};

builder.Logging
    // Add the OpenTelemetry as logging output
    .AddOpenTelemetry(logger =>
    {
        // The resources identify our application
        logger.SetResourceBuilder(ResourceBuilder.CreateDefault().AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: serviceInstanceId));
        logger.IncludeScopes = true;
        logger.IncludeFormattedMessage = true;
        // Make OTLP exporter out of it
        logger.AddOtlpExporter(local);
    });

builder.Services
    // Add the OpenTelemetry
    .AddOpenTelemetry()
    // With common resource configuration
    .ConfigureResource(resource =>
        resource.AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: serviceInstanceId))
    // With tracing
    .WithTracing(tracing =>
    {
        tracing.AddHttpClientInstrumentation();
        // Make OTLP exporter out of it
        tracing.AddOtlpExporter(local);
    })
    // And metrics
    .WithMetrics(metrics =>
    {
        metrics.AddRuntimeInstrumentation();
        metrics.AddHttpClientInstrumentation();
        metrics.AddMeter("Microsoft.AspNetCore.Hosting");
        metrics.AddMeter("Microsoft.AspNetCore.Server.Kestrel");
        // Make OTLP exporter out of it
        metrics.AddOtlpExporter(local);
    });
```

That's the bare minimum required to have OpenTelemetry in a .NET application.

#### ActivitySource
The `ActivitySource` comes from the `System.Diagnostics.Activity` from the .NET 5 framework. The concept of the `ActivitySource` is equal to the `Trace` in OpenTelemetry. It's basically a component responsible for creating `Activity` that is equivalent to the OpenTelemetry `Span`. The `Activity` tracks the application behaviour.

It's recommended to have a single `ActivitySource` per library or create another one if you expect that you or somebody else may want to turn on or off telemetry sources independently. The `source` of the activity must be unique and the `version` is an optional parameter, but recommended to use.

```csharp
public static class AppActivitySource
{
    public static readonly string ActivitySourceName = "SampleApp";
    
    public static readonly ActivitySource ActivitySource = new(ActivitySourceName, "dev");
}
```

The activity you simply create with `StartActivity()` within the `using` block that makes the `Activity` boundary. Be aware that `StartActivity()` may return a `null` value instead of an object if there are no listeners for the `ActivitySource`. You can also attach a tag to the activity by using `activity.SetTag(AppTags.MyTag, value)`.

Every `ActivitySource` **must be registered** in the OpenTelemetry tracing configuration by its name.

```csharp
builder.Services
    .AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddHttpClientInstrumentation();
        // register the activity source
        tracing.AddSource(AppActivitySource.ActivitySourceName);
        tracing.AddOtlpExporter(exporter);
    });
```

#### Create custom metrics
To create custom metrics you just need to create a class that uses `IMeterFactory` to create a meter to record counter, up-down counter or histogram values.

```csharp
// Register in the DI as singleton
public class SampleMetrics
{
    // As the name we usually use the name of the assembly.
    public const string METER_NAME = "SampleApp";
    
    private readonly Histogram<double> _averageTemperatureHistogram;
    
    public SampleMetrics(IMeterFactory factory)
    {
        var meter = factory.Create(METER_NAME);
        // The metrics names are usually lowercase and beginning with the name of meter
        _averageTemperatureHistogram = meter.CreateHistogram<double>("sampleapp.average_temperature");
    }

    public void RecordAverageTemperature(double temperature)
    {
        _averageTemperatureHistogram.Record(temperature);
    }
}
```

Finally, similarly to the `ActivitySource`, we have to register the meter namespace to be able to reach them to the collector.

```csharp
builder.Services
    .AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddRuntimeInstrumentation();
        metrics.AddHttpClientInstrumentation();
        metrics.AddMeter("Microsoft.AspNetCore.Hosting");
        metrics.AddMeter("Microsoft.AspNetCore.Server.Kestrel");
        // Register the new meter, be caution you have to use the assembly name
        metrics.AddMeter("SampleApp");
        metrics.AddOtlpExporter(local);
    });
```

#### Serilog
The Serilog is one of the most well-known and used libraries in the .NET ecosystem. Depending on how you use the Serilog you may end up being unable to see logs in the Dashboard. The configuration that I am using is the following.

```csharp
// Create the Serilog configuration
var serilog = new LoggerConfiguration();
serilog
    // Load from appsettings.json or set manually
    .MinimumLevel.Information()
    .WriteTo.Console()
    // Because we set the Serilog as the logging provider we have to use
    // the OpenTelemetry sink
    .WriteTo.OpenTelemetry(options =>
    {
        options.Protocol = OtlpProtocol.Grpc;
        options.Endpoint = "http://localhost:4317";
        // We have to add resource attributes to correctly identify our app
        options.ResourceAttributes = new Dictionary<string, object>
        {
            { "service.name", serviceName },
            { "service.version", serviceVersion },
            { "service.instance.id", serviceInstanceId },
        };
    });

// Set the Serilog as the logging provider
builder.Services.AddSerilog(serilog.CreateLogger());
```

### Exporters
In the sample I've used Aspire for local development, which is great and gives us developers the overview that we need before we push the app to the production, but what about the production environment? Thanks to the OpenTelemetry abstraction we can use any Observability provider out there, it's simple as literally the single change in configuration that allow you to use production-ready solution like [OneUptime](https://oneuptime.com/).

Here is the configuration for [OneUptime](https://oneuptime.com/).

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:
    verbosity: detailed

  # Exportet over HTTP
  otlphttp:
    endpoint: "https://oneuptime.com/otlp"
    # Requires use JSON encoder insted of default Proto(buf)
    encoding: json
    headers: 
      "Content-Type": "application/json"
      "x-oneuptime-token": "---" # Your OneUptime token

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlphttp]
    metrics:
      receivers: [otlp]
      exporters: [otlphttp]
    logs:
      receivers: [otlp]
      exporters: [debug, otlphttp]
```

That's it.

---

August, 2024 <br />
[Sebastian](mailto:ja@sebastianbusek.cz) <br />