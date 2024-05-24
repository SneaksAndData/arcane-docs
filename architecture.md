# Arcane Operator Architecture overview

## Introduction

Arcane Operator consists of several services that watch for changes in a Kubernetes cluster and perform changes based
on the desired cluster configuration described in the set of Custom Resources (CRs) that it manages.

Each `*OperatorService` class is responsible for managing a specific CR type. For example, `StreamClassOperatorService`
manages the `StreamClass` resources and `JobOperator` manages `Job` resources etc.

Every operator class follows the following pattern:
- Watches for changes in the cluster using an Akka Source object that produces instances of `ResourceEvent` class.
- Makes decisions on what changes in internal state or cluster should be made based on the event type.
- Produces a list of `Command` objects enriched with context available in the moment of the event.
- Pushes the commands to a `CommandHandler` that retrieves additional context (if required) and executes the command.
- A command handler may produce additional commands and invoke other handlers.

![architecture diagram](architecture.pdf "Arcane Operator Architecture Diagram")

## Components
### Event-based services:
- `StreamClassOperatorService` - manages `StreamClass` resources.
   Upon creating a new object with the `StreamClass` kind, it creates a new Akka Source that watches for events
   related to resource described in the `StreamClass` object and holds the kill switches attached to the source.
   If the `StreamClass` object is deleted, the source is stopped and Arcane stops watching for events related to the 
   resource. Existing stream definitions will not be deleted and jobs started from those definitions will continue to run.

- `StreamOperatorService` - manages `StreamDefinition` resources
  (`SqlServerChangeTrackingStream`, `RestApiDynamicAuth`,`RestApiFixedAuth`, etc).
  When a new stream definition is created, `StreamOperatorService` creates new Kubernetes job, converts the
  StreamDefinition object into a job configuration and submits it to the Kubernetes API.
  If the Stream definition was modified Arcane immediately stops the existing job and creates a new one with the updated
  configuration. The `StreamOperatorService` does not handle deletion of the stream definitions. Each stream job
  contains an owner reference to the stream definition object and when stream definition is deleted, it should be
  handled by the Kubernetes garbage collector.

- `StreamingJobOperatorService` - handles events related to the Kubernetes `Job` objects. If the job is deleted or
   modified, the operator reads the `StreamDefinition` object and creates a new job, but only if stream definition is
   still present and not suspended. If the streaming job is in a failed state, the operator sets the status of the stream
   definition to `CrashLoopDetected` and stops listening for events related to the StreamDefinition object.

### Commands
All commands in Arcane are inherited from the `KubernetesCommand` class. Command classes are responsible for
holding the context required to execute the command. For example, `SetAnnotationCommand` holds the affected
resource object and key-value pair of the annotation that should be set.


### Command Handlers
Command handlers are responsible for executing the commands. Each command handler is responsible for a specific type of
command. The command handlers can produce additional commands and invoke other handlers.

Also the command handlers can retrieve additional context required to execute the command. For example, the
handlers that handles commands that affects the StreamDefinition object may need to retrieve the StreamClass object
to read the stream definition metadata.

