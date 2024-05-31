# How to develop an Arcane Streaming plugin

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

This schema is required for the Arcane operator to be able to manage the plugin lifecycle. Every custom resources that
defines stram must follow
this convention until the  [StreamClass-based contracts](https://github.com/SneaksAndData/arcane-framework/issues/23)
feature is implemented.

## Sources and Sinks

Arcane provides a set of Akka.net-based sources and sinks in the `Arcane.Framework` package. If a stream plugin needs 
a source of sink that is ot implemented in the framework, the plugin developer should add it to the framework package to
allow other plugins to use it.

## Arcane.Framework hosting extensions
Arcane.Framework provides a set of extension methods for the `IHostBuilder` that can simplify the streaming job development.

### Using with Datadog services
Developer may use the `AddDatadogLogging` extension method to add Datadog logging to the streaming job. This method uses the `Serilog` library to log messages to Datadog.
Additionally this method adds enrichers that enrich the log messages with the `streamId` and `streamKind` properties.

```csharp
    var hsot = Host.CreateDefaultBuilder(args).AddDatadogLogging();
```

## Stream Context and StreamContextWriter

The Arcane framework provides a `IStreamContext` interface that should be used by the runner plugin to read the
stream configuration and stream metadata fields like `StreamId`, `StreamKind` and `IsBackfilling`.

The stream plugin should implement `IStreaContext` and `IStreamContextWriter` interfaces to allow the framework to
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
2. Arcane Operator removes all properties that are marked as `secretRefs` in the `StreamClass` resource of the stream plugin.
3. Arcane Operator serializes the `JsonElement` object to the JSON string and sets it to the `STREAMCONTEXT__SPEC` environment variable.

**NOTE**: Operator does not interpret neither the `spec` nor the `secretRefs` field in any way besides the described above.


### Implementing the data stream

Arcane Framework provides an `IStreamGraphBuilder` interface that should be implemented by every stream plugin.
The stream plugin should implement this interface to define the stream graph. The built-in stream runner requires the
stream graph to be materialized in to pair `(UniqueKillSwitch, Task)` to allow proper stream lifecycle management.
The example implementation of the `IStreamGraphBuilder` interface:
```csharp
/*
   Define a RestApiGraphBuilder class that implements the IStreamGraphBuilder interface and
   consumes the SomeApiStreamContext class as a context
*/
public class RestApiGraphBuilder: IStreamGraphBuilder<SomeApiStreamContext>
{
    public IRunnableGraph<(UniqueKillSwitch, Task)> BuildGraph(SomeApiStreamContext context)
    {
        // Create a source using the stream context
        var source = this.GetSource(configuration)
        
        // Get a stream metrics dimensions from the source tags
        var dimensions = source.GetDefaultTags().GetAsDictionary(context, context.StreamId);
        
        // Create a sink using the stream context
        var sink = this.GetSink(configuration, source.GetParquetSchema(), this.blobStorageWriter, this.sinkLocation);
        
        // The actual runnable graph definition
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

### Typed and untyped stream builder provider
The `IStreamGraphBuilder` interface is a generic interface parameterized by the stream context type. The
recommended approach is use strongly typed stream context classes to allow the stream plugin to access the stream
context without type casting:

```csharp
    // Create Host builder
    var host = Host.CreateDefaultBuilder(args)
    
         // Configure datadog logging
        .AddDatadogLogging( (_, _, configuration) => configuration.WriteTo.Console())
        
         // Inject the stream graph builder and the stream context
        .ConfigureRequiredServices(services => services.AddStreamGraphBuilder<RestApiGraphBuilder, RestApiDynamicAuthStreamContext>())
        
         // Configure additional services
        .ConfigureAdditionalServices((services, context) =>
         {
             // Add additional services here
         })
    .Build()
```
But for dynamic stream plugins that require the stream context to be defined at runtime, the `IStreamGraphBuilder` can be injected as an untyped service:
```csharp
    // Create Host builder
    var host = Host.CreateDefaultBuilder(args)
    
         // Configure datadog logging
        .AddDatadogLogging( (_, _, configuration) => configuration.WriteTo.Console())
        
         // Inject the stream graph builder and the stream context
          .ConfigureRequiredServices(services =>
          {
              return services.AddStreamGraphBuilder<RestApiGraphBuilder>(hostBuilderContext =>
              {
                  // Dynamically load the stream context based on the stream kind
                  // This logic is used to determine the stream context type based on the stream kind
                  // Until the StreamComponent is implemented, this .
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
            ContextType2 configuration2 => this.BuildGraph(configuration1),
            CotnextType3 configuration3 => this.BuildGraph(configuration1),
            _ => throw new ArgumentOutOfRangeException($"Unsupported stream context type: {context.GetType()}")
        };
    }
```


## Streaming job lifecycle and permissions required for the stream runner in the cluster
The communication between the stream runner and the Arcane operator is done in two ways:
1. The Arcane Operator creates a streaming job in the Kubernetes cluster and provides the stream configuration
   through the environment variables.
2. The stream runner can finish with the `Success` status and in this case it will be restarted in the streaming mode.
3. The stream runner can finish with the `Failed` status and in this case the Operator will create the
   `arcane.streaming.sneaksanddata.com/state: crash-loop`  annotation on the stream definition and stop processing the
   SD events until operator removes this annotation manually.
4. The stream runner may annotate the streaming job with the `arcane.streaming.sneaksanddata.com/state: schema-mismatch`
   annotation and exit with the `Success` status. In this case the operator restart stream with the `backfill` mode
   and remove `arcane/state: schema-mismatch` annotation from the stream definition object.

The stream runner requires the permissions to patch V1Job objects in the namespace where the stream is running.

The stream operator requires the following permissions:
- Patch the stream definition objects in namespace where stream jobs are running
- Set status of the stream definition objects in the namespace where stream jobs are running.
- Patch the stream definition objects in the namespace where stream jobs are running

It's a stream plugin responsibility to provide the correct permissions for the actions described above.

## Defining the SteamClass resource

Every stream plugin should define a `StreamClass` resource for each stream definition that it installs.
The stream class should contain at least the following fields:

- `apiGroupRef`: Name of th API group that the stream plugin belongs to, e.g. `streaming.sneaksanddata.com`
- `kindRef`: Name of th API group that the stream plugin belongs to, e.g. `RestApiFixedAuth`
- `apiVersion`: API version of the stream definition managed by the stream class, e.g. `v1beta1`
- `pluralName`: Plural name of the stream definition objects, e.g. `api-fas`
- `secretRefs`: List of the secret references that the stream plugin uses to store the sensitive (if any).

## Advanced features
### Advanced stram runner lifecycle management
The `IStreamLifetimeService` interface is intended to provide ability to stop the stream gracefully and to handle
the Kubernetes pod lifecycle events. By default it is implemented by the `StreamLifetimeService` class that triggers
the graph KillSwitch when the pod receives the SIGTERM signal.

If the stream plugin requires additional logic to handle the stream lifecycle, it may implement the
`IStreamLifetimeService` interface and inject it to the stream runner using the optional `addStreamLifetimeService`
parameter of the `ConfigureRequiredServices` extension method.

This service is not intended to be called by the stream plugin code.

### Extending the stream runner
The stream runner is a class that implements the `IStreamRunnerService` interface and is responsible for the stream
execution. The stram runner provided the framework materializes the streaming graph returned by the StreamGraph Builder
and holds the kill switch that allows to stop the stream gracefully.

A custom implementation of the `IStreamRunnerService` interface can be provided to extend the stream runner
functionality using the `addStreamRunnerService` parameter of the `ConfigureRequiredServices` extension method.


### Custom Exception handling
The framework provides the `ArcaneExceptionHandler` class that is used to handle the exceptions thrown by the stream.
This class handles the exceptions derived by the `SchemaException` in the following way:

- In case of `SchemaMismatchException` the stream runner reports the schema mismatch to the operator and stops the
  stream with the `SUCCESS` exit code. That triggers the operator to restart the stream in the backfill mode.
 
- In case of `SchemaInconsistentException` the stream runner exits with the `RETRY` exit code and expects that the
 Job will restart the straam without incrementing the failure counter.

Any additional exceptions should be handled by the stream plugin code or using the function provided as value
using the `handleUnknownException` parameter of the `RunStream` extension method. This function must return
the exit code for the stream runner.

Any unhandled exceptions will be logged and the stream runner will exit with the `FAILED` exit code.

## Examples

Example of the Program.cs for the plugin:

```csharp
// Create the bootstrap logger using serilog and the default logging provider
Log.Logger = DefaultLoggingProvider.CreateBootstrapLogger(nameof(Arcane));

// Application exit code
int exitCode;
try
{
    // Create the host builder
    exitCode = await Host.CreateDefaultBuilder(args)
        // Configure datadog logging
        .AddDatadogLogging( (_, _, configuration) => configuration.WriteTo.Console())
        // Configure required services
        .ConfigureRequiredServices(services =>
        {
            services.AddStreamGraphBuilder<RestApiGraphBuilder, RestApiDynamicAuthStreamContext>();
            services.AddSingleton<IStreamContextWriter, RestApiDynamicAuthStreamContext>();
        })
        // Configure additional services
        .ConfigureAdditionalServices((services, context) =>
         {
            services.AddAzureBlob(AzureStorageConfiguration.CreateDefault());
            services.AddDatadogMetrics(configuration: DatadogConfiguration.UnixDomainSocket(context.ApplicationName));
            services.AddSingleton((IAmazonS3) new AmazonS3Client());
            services.AddSingleton<IBlobStorageWriter, AmazonBlobStorageWriter>();
         })
    // Build the console application host
    .Build()
    // Run the stream
    .RunStream(Log.Logger);
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

return exitCode;
```
