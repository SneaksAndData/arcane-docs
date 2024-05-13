# Concepts

The Arcane platform consists of the following components, which can be installed using Helm charts:

- Arcane Operator that manages the lifecycle of the streaming jobs
- Arcane Stream Plugins: REST API, Microsoft SQL Server, Microsoft CDM change feed, and more to be added in the future.

During the installation Arcane Operator deploys the following CRDs:
- `StreamClass`, which is used to define configuration of the stream plugin.
- `StreamingJobTemplate`, which is used to generate a Job object for the Kubernetes.

The configuration of each individual streaming job is managed by the custom resources installed
by the stream plugin Helm Chart. These  resources referred to as `StreamDefinitions` in the text below.

During the stream plugin installation process the
helm chart installs the `StreamClass` object that contains the `StreamDefinition` API group, version and plural name.

Also, the plugin Helm chart may install the optional `StreamingJobTemplate` object, that contains the
Kubernetes Job definition that the Operator uses to generate streaming jobs.

## Stream reliability
At the beginning of the streaming job, it fetches a small volume of historical data to ensure that the
stream fetches all data, even in the case that the streaming job was restarted for some reason.
The depth of the historical data is controlled by the lookBackInterval parameter in the StreamDefinition object.

## Backfilling and schema evolution
Currently, supported stream plugins use a polling mechanism to fetch data from the source. If the source supports
schema evolution and the source data schema has changed, the stream plugin
completes and requests the Arcane Operator to backfill the data.

The data backfill process is implemented by the Arcane Operator by creating a new job with
an (potentially) different job template and providing the `STREAMCONTEXT__BACKFILL` environment
variable to the streaming job.

After the backfill is completed, Arcane Operator resumes the streaming job from the point where it was stopped.

## Secret Management for streaming jobs
Arcane Operator mounts the Kubernetes secrets to the streaming job pods as environment variables.
Each `StreamDefinition` object may contain a reference to a Kubernetes
secret that should be serializable to the `V1SecretEnvSource` object.

The list of the `StreamDefinition` fields that should be serialized to a V1SecretEnvSource object
defined in the `StreamClass` object managed by the stream plugin Helm chart.
