# Arcane Streaming Plugin Developer Guide

The Arcane plugin consists of three main components:
1. The stream runner which implements an akka stream that extracts data using an Akka.NET source, transforms it and
   writes data to a single sink. The output data format depends on the sink used to write data.
2. One or more Custom Resource Definitions (CRDs) that defines stream source settings.
3. The Kubernetes object of the `StreamClass` kind that is used by the Arcane Operator to handle Stream CRDs events.

# Implementing the Stream Runner 

## Sources and Sinks

Arcane provides a set of Akka.NET-based sources and sinks in the `Arcane.Framework` package. If a stream plugin needs
a source of sink that is not implemented in the framework, the plugin developer should add it to the framework package to
allow other plugins to use it. You can explore the available sinks and sources by referring to the
`Arcane.Framework.Sinks` and `Arcane.Framework.Sources` namespaces respectively.

## Stream Context and StreamContextWriter

The Arcane framework provides a `IStreamContext` interface that should be used by the runner plugin to read the
stream configuration and stream metadata fields like `StreamId`, `StreamKind` and `IsBackfilling`.

The stream plugin should implement `IStreamContext` and `IStreamContextWriter` interfaces to allow the framework to
initialize the plugin metadata, logs and metrics properly:
```csharp
    public interface IStreamContext
    {
        // Required stream metadata properties provided by operator through environment variables
        public string StreamId { get; }
        public string StreamKind { get; }
        public bool IsBackfilling { get; }
        
        // Stream configuration properties (for example, REST Api configuration)
        public string AuthUrl { get; private set; }
        public string TokenPropertyName { get; private set; }
    }

    public interface IStreamContextWriter
    {
        void SetStreamId(string streamId);
        void SetStreamKind(string streamKind);
        void SetIsBackfilling(bool isBackfilling);
    }
```

The `StreamId` and `StreamKind` properties are required for the framework to properly manage the stream lifecycle.
The `IsBackfilling` property is required to allow the plugin to distinguish between the backfilling and the regular
streaming mode.

Any additional configuration properties are retrieved from the environment variable `STREAMCONTEXT__SPEC` using the
JSON deserializer.

The value of the `STREAMCONTEXT__SPEC` environment variable is generated from the stream custom resource `spec` field
in the following way:
1. Arcane Operator deserializes the `spec` field to the `JsonElement` object.
2. Arcane Operator removes all properties that are marked as `secretRefs` in the `StreamClass` resource of the stream 
   plugin.
3. Arcane Operator serializes the `JsonElement` object to the JSON string and sets it to the `STREAMCONTEXT__SPEC`
   environment variable.

**NOTE**: Operator does not interpret either the `spec` or `secretRefs` fields in any way besides the described above.

## Implementing the Data Stream

Arcane Framework provides an `IStreamGraphBuilder` interface that should be implemented by every stream plugin.
The stream plugin should implement this interface to define the stream graph. The built-in stream runner requires the
stream graph to be materialized in to pair `(UniqueKillSwitch, Task)` to allow proper stream lifecycle management.
The example implementation of the `IStreamGraphBuilder` interface:
```csharp
/*  
   Here, we define a RestApiGraphBuilder class that implements the IStreamGraphBuilder interface and  
   uses the SomeApiStreamContext class as a context  
*/  
public class RestApiGraphBuilder: IStreamGraphBuilder<SomeApiStreamContext>
{
    public IRunnableGraph<(UniqueKillSwitch, Task)> BuildGraph(SomeApiStreamContext context)
    {
        // First, a source is created using the stream context  
        var source = this.GetSource(configuration)
        
        // Next, we get stream metrics dimensions from the source tags
        var dimensions = source.GetDefaultTags().GetAsDictionary(context, context.StreamId);
        
        // Then, a sink is created using the stream context
        var sink = this.GetSink(configuration, source.GetParquetSchema(), this.blobStorageWriter, this.sinkLocation);
        
        // Finally, we define the actual runnable graph
        return Source.FromGraph(source)
            .GroupedWithin(rowsPerGroup, groupingInterval)
            .Select(grp =>
            {
                var rows = grp.ToList();
                metricsService.Increment(DeclaredMetrics.ROWS_INCOMING, dimensions, rows.Count);
                return rows;
            })
            .Log(context.StreamKind)
            .ViaMaterialized(KillSwitches.Single<List<JsonElement>>(), Keep.Right)
            .ToMaterialized(sink, Keep.Both);
    }
}
```

## Arcane.Framework hosting extensions
Arcane.Framework provides a set of extension methods for the `IHostBuilder` that can simplify the streaming job
development. To create a streaming console application, developers should use these extensions in order to build the
Dependency Injection container service provider that contains all required services. These extensions are described below.

### Using With Datadog Services

#### Logging 

Arcane comes with telemetry services provider that use the `Serilog` logging library and datadog metrics facilities.
In order to use the DataDog logger, developers may use the `AddDatadogLogging` extension method to add Datadog logging to the
streaming job. This injects a serilog logging provider into the Actor system and the DataDog client.
```csharp
using Arcane.Framework.Providers.Hosting;

var host = Host.CreateDefaultBuilder(args).AddDatadogLogging();
```

#### Metrics

The Arcane framework itself does not rely on any particular metrics libraries, but stream plugin developers 
may consume the `MetricsService` available in the [Snd.Sdk package](https://github.com/SneaksAndData/esd-services-sdk/blob/76a7bfe242a27d587363a65a538461192b5f0532/src/Metrics/Base/MetricsService.cs#L11)
to publish metrics using DogStatsD or any other supported protocol.

The metrics provider can be injected using the `ConfigureAdditionalServices` extension method:
```csharp
using Arcane.Framework.Providers.Hosting;
using Snd.Sdk.Metrics.Providers;

var host = Host.CreateDefaultBuilder(args)
           .AddDatadogLogging()
           .ConfigureAdditionalServices((services, context) =>
            {
                services.AddDatadogMetrics(DatadogConfiguration.UnixDomainSocket(context.ApplicationName));
            });
```

For DataDog configuration details please consider the Arcane Configuration manual.

### Injecting the IStreamGraphBuilder
The `IStreamGraphBuilder` interface is a generic interface parameterized by the stream context type. The recommended
approach is to use strongly typed stream context classes to allow the stream plugin to access the stream context without
type casting:
```csharp
// Create a Host Builder
var host = Host.CreateDefaultBuilder(args)

     // Inject the stream graph builder and the stream context
    .ConfigureRequiredServices(services => services.AddStreamGraphBuilder<RestApiGraphBuilder, RestApiDynamicAuthStreamContext>())
    
     // Configure additional services
    .ConfigureAdditionalServices((services, context) =>
     {
         // Add additional services here
     })
.Build()
```

But for dynamic stream plugins that require the stream context to be defined at runtime, the `IStreamGraphBuilder` can
be injected as an untyped service:
```csharp
// Create a Host Builder
var host = Host.CreateDefaultBuilder(args)
    
      // Inject the stream graph builder and the stream context
      .ConfigureRequiredServices(services =>
      {
          return services.AddStreamGraphBuilder<RestApiGraphBuilder>(hostBuilderContext =>
          {
              // Dynamically load the stream context based on the stream kind
              // This logic is used to determine the stream context type based on the stream kind
              var streamContext = GetStreamContext(hostBuilderContext);
              return streamContext;
          });
     })
     
     // Configure additional services
    .ConfigureAdditionalServices((services, context) =>
     {
         // Add additional services here
     })
.Build()
```

In this case the `StreamGraphBuilder` should accept `IStreamContext` as a parameter and cast it to the actual stream context type:
```csharp
    public IRunnableGraph<(UniqueKillSwitch, Task)> BuildGraph(IStreamContext context)
    {
        return context switch
        {
            ContextType1 configuration1 => this.BuildGraph(configuration1),
            ContextType2 configuration2 => this.BuildGraph(configuration2),
            CotnextType3 configuration3 => this.BuildGraph(configuration3),
            _ => throw new ArgumentOutOfRangeException($"Unsupported stream context type: {context.GetType()}")
        };
    }
```

# Defining the StreamDefinition CRD
Each Arcane plugin should have a Helm chart that has a Custom Resource definition that allows users to configure the plugin.
The CRD must be namespaced and provide the following fields:
- `spec`: The configuration of the plugin.
- `status`: The status of the plugin. The status layout **must** follow the following schema:

```yaml
status:
  type: object
  properties:
    phase:
      type: string
      enum:
        - RESTARTING
        - RUNNING
        - RELOADING
        - TERMINATING
        - STOPPED
        - SUSPENDED
        - FAILED
    conditions:
      type: array
      items:
        type: object
        required:
          - status
          - type
        properties:
          message:
            type: string
          type:
            type: string
            enum:
              - WARNING
              - ERROR
              - INFO
              - READY
          status:
            type: string
            enum: 
              - "True"
              - "False"
```

This schema is required for the Arcane operator to manage the plugin lifecycle.
Every custom resource that defines a stream must follow this convention until the
[StreamClass-based contracts](https://github.com/SneaksAndData/arcane-framework/issues/23) feature is implemented.

## Permissions Required For the Stream Runner In The Cluster
The stream runner requires permission to patch Job objects in the namespace where the stream is running.

The stream operator requires the following permissions:

- To patch the stream definition objects in the namespace where stream jobs are running.
- To patch the status of the stream definition objects in the same namespace.

Providing the permissions required for the actions described above is responsibility of Helm Chart for the stream plugin.

# Defining the SteamClass Resource

Every stream plugin should define a `StreamClass` resource for each stream definition that it installs.
The stream class should contain at least the following fields:

- `apiGroupRef`: Name of the API group that the stream plugin belongs to e.g., `streaming.sneaksanddata.com`.
- `kindRef`: Name of the object kind of the stream definition e.g., `RestApiFixedAuth`.
- `apiVersion`: API version of the stream definition managed by the stream class e.g., `v1beta1`.
- `pluralName`: Plural name of the stream definition objects e.g., `api-fas`.
- `secretRefs`: List of the secret references that the stream plugin uses to store the sensitive data (if any).

# Advanced Features
### Advanced stream runner lifecycle management
The `IStreamLifetimeService` interface is intended to provide the ability to stop the stream gracefully and to handle
the Kubernetes pod lifecycle events. By default, it is implemented by the `StreamLifetimeService` class that triggers
the graph KillSwitch when the pod receives the SIGTERM signal.

If the stream plugin requires additional logic to handle the stream lifecycle, it can implement the
`IStreamLifetimeService` interface and inject it into the stream runner using the optional `addStreamLifetimeService`
parameter of the `ConfigureRequiredServices` extension method.

This service is not intended to be called by the stream plugin code.

### Extending the Stream Runner
The stream runner is a class that implements the `IStreamRunnerService` interface and is responsible for the stream
execution. The stream runner provided by the framework materializes the streaming graph returned by the StreamGraph Builder
and holds the kill switch, allowing it to stop the stream gracefully.

A custom implementation of the `IStreamRunnerService` interface can be provided to extend the stream runner
functionality using the `addStreamRunnerService` parameter of the `ConfigureRequiredServices` extension method.

### Custom Exception Handling
The framework provides the `ArcaneExceptionHandler` class that is used to handle the exceptions thrown by the stream.
This class handles the exceptions derived from the `SchemaException` in the following way:

- In the case of a `SchemaMismatchException` the stream runner reports the schema mismatch to the operator and stops the
  stream with the `SUCCESS` exit code. That triggers the operator to restart the stream in the backfill mode.
 
- In the case of a `SchemaInconsistentException` the stream runner exits with the `RETRY` exit code and expects that the
 Job will restart the stream without incrementing the failure counter.

Any additional exceptions should be handled by the stream plugin code or using the function provided as value
with the `handleUnknownException` parameter of the `RunStream` extension method. This function must return
the exit code for the stream runner.

Any unhandled exceptions will be logged and the stream runner will exit with the `FAILED` exit code.

# Examples

Example of the Program.cs for the plugin:

```csharp
// First, we create a serilog bootstrapp logger.
Log.Logger = DefaultLoggingProvider.CreateBootstrapLogger(nameof(Arcane));

// Next, we create a variable for application exti code.
int exitCode;
try
{
    // Next, we create the HostBuilder.
    var hostBuilder = Host.CreateDefaultBuilder(args)
    
         // Configure datadog logging.
        .AddDatadogLogging( (_, _, configuration) => configuration.WriteTo.Console())
        
         // Configure the required services (graph builder and stream context).
        .ConfigureRequiredServices(services =>
        {
            services.AddStreamGraphBuilder<RestApiGraphBuilder, RestApiDynamicAuthStreamContext>();
        })
        
         // Configure additional services (metrics and AWS S3 writer for Sink output).
        .ConfigureAdditionalServices((services, context) =>
         {
            services.AddDatadogMetrics(DatadogConfiguration.UnixDomainSocket(context.ApplicationName));
            services.AddAwsS3Writer(AmazonStorageConfiguration.CreateFromEnv());
         })
        
    // Build the console application host.
    var appHost = hostBuilder.Build();
    
    // Finally, we can run the stream and wait for completion
    exitCode = await.RunStream<RestApiDynamicAuthStreamContext>(Log.Logger);
}
catch (Exception ex)
{
    // Handle exception that can occur during the host initialization
    Log.Fatal(ex, "Host terminated unexpectedly");
    return ExitCodes.FATAL;
}
finally
{
    // Close the bootstrap logger
    await Log.CloseAndFlushAsync();
}

// Return the exit code and complete the application.
return exitCode;
```
