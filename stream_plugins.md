# How to develop an Arcane Streaming plugin

Each Arcane plugin should have a Helm chart that has a Custom Resource definition that allows users to configure the plugin.
The CRD must be namespaced and provide the follwing fields:
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

### Typeed and untyped stream builder provider

## Required services and permissions required for the stream runner in the cluster

## Defining the SteamClass resource

## Additional services and sink configuration

## Advaned features

### Ability to implmenet IStreamLifetimeService interface

### Ability to implmenet IStreamRunnerService

### Ability to implement IArcaneExceptionHandler

##

Example of the Program.cs for the plugin:

```csharp
```
