# Getting started

## Installation

Arcane consists of two main components:

- Stream operator service
- Stream plugin

All components can be installed using Helm charts. To be able to run Arcane streams you should install the stream
operator and at least one stream plugin.


The default installation options manage CRDs and RBAC resources for both the stream operator and stream plugins.
If you do not wish to install the CRDs, you can disable them by setting `customResourceDefinitions.create` to `false`.

If the custom resource definitions installation is disabled, the operator will expect the CRDs to be installed
in the cluster prior to the operator installation.

To install Arcane using Helm, you should use Helm version `3.8.0` or above.
For the versions prior to `3.8.0`, you should use the `HELM_EXPERIMENTAL_OCI=1` environment variable or refer
to the [Helm documentation](https://helm.sh/docs/topics/registries/) for the OCI-based registries.

### Stream operator

Run the following command:

```bash
# Create a namespace for the operator installation
$ kubectl create namespace arcane

# Install the operator in the created namespace
$ helm install arcane oci://ghcr.io/sneaksanddata/helm/arcane-operator \
  --version v0.0.0-41-g4e59a38 \
  --namespace arcane
```

This command creates a namespace `arcane` and installs the stream operator in it. By default, the Helm chart installs two CRDs:
- StreamClass
- StreamingJobTemplate

The resources of both kinds are being installed by the streaming plugins.

#### Verify the installation

To verify the operator installation run the following command:

```bash
$ kubectl get pods -l app.kubernetes.io/name=arcane-operator --namespace arcane
```

It should produce the similar output:

```bash
NAME                               READY   STATUS    RESTARTS   AGE
arcane-operator-55988bbfcb-ql7qr   1/1     Running   0          25m
```

### Streaming plugins

To install a REST API streaming plugin, run the following command:

```bash
# Install the REST API streaming plugin in the namespace created in the previous step
$ helm install arcane-stream-rest-api oci://ghcr.io/sneaksanddata/helm/arcane-stream-rest-api \
  --version v0.0.0-20-gf44a1b4 \
  --set jobTemplateSettings.extraEnvFrom[0].secretRef.name=azure-connection-string \
  --namespace arcane
```

#### Verify the installation

To verify the plugin installation run the following command:

```bash
$ kubectl get stream-classes --namespace arcane -l app.kubernetes.io/name=arcane-stream-rest-api
```

It should produce the similar output:
```bash
NAME                              PLURAL     APIGROUPREF                   PHASE
arcane-stream-rest-api-fa         api-fas    streaming.sneaksanddata.com   READY
arcane-stream-rest-api-paged-da   api-pdas   streaming.sneaksanddata.com   READY
arcane-stream-rest-api-paged-fa   api-pfas   streaming.sneaksanddata.com   READY

```

When the stream classes are transitioned to the `READY` phase, the plugin is ready to be used.

Additionally, stream plugin may install a streaming job template.
To verify the streaming job template installation run the following command and check the output:

```bash
$ kubectl get streaming-job-templates --namespace arcane
NAME                     AGE
arcane-stream-rest-api   12m
```

### Stream Definitions

When the stram class installation is completed, you can create a stream definition.

This example creates a stream definition for the REST API plugin that fetches air quality data from the OpenAQ API and
stores it in the Azure Blob Storage in multiline JSON format.



Create a secret with the Azure Blob Storage connection string:

```bash
kubectl create secret generic azure-connection-string \
  --namespace arcane  \
  --from-file=AZURE_STORAGE_CONNECTION=./secret-data.txt
```

Create a stream definition for the OpenAQ API:

```yaml
apiVersion: streaming.sneaksanddata.com/v1beta1
kind: RestApiPagedFixedAuth
metadata:
  name: openaq-air-quality
  namespace: arcane
spec:
  apiSchemaEncoded: eyJwcm9wZXJ0aWVzIjp7InJlc3VsdHMiOnsiaXRlbXMiOnsicHJvcGVydGllcyI6eyJjaXR5Ijp7InR5cGUiOiJzdHJpbmcifSwiY29vcmRpbmF0ZXMiOnsicHJvcGVydGllcyI6eyJsYXRpdHVkZSI6eyJ0eXBlIjoibnVtYmVyIn0sImxvbmdpdHVkZSI6eyJ0eXBlIjoibnVtYmVyIn19LCJ0eXBlIjoib2JqZWN0In0sImNvdW50cnkiOnsidHlwZSI6InN0cmluZyJ9LCJkYXRlIjp7InByb3BlcnRpZXMiOnsibG9jYWwiOnsidHlwZSI6InN0cmluZyJ9LCJ1dGMiOnsidHlwZSI6InN0cmluZyJ9fSwidHlwZSI6Im9iamVjdCJ9LCJlbnRpdHkiOnsidHlwZSI6InN0cmluZyJ9LCJpc0FuYWx5c2lzIjp7InR5cGUiOiJib29sZWFuIn0sImlzTW9iaWxlIjp7InR5cGUiOiJib29sZWFuIn0sImxvY2F0aW9uIjp7InR5cGUiOiJzdHJpbmcifSwibG9jYXRpb25JZCI6eyJ0eXBlIjoiaW50ZWdlciJ9LCJwYXJhbWV0ZXIiOnsidHlwZSI6InN0cmluZyJ9LCJzZW5zb3JUeXBlIjp7InR5cGUiOiJzdHJpbmcifSwidW5pdCI6eyJ0eXBlIjoic3RyaW5nIn0sInZhbHVlIjp7InR5cGUiOiJudW1iZXIifX0sInR5cGUiOiJvYmplY3QifSwidHlwZSI6ImFycmF5In19LCJ0eXBlIjoib2JqZWN0In0=
  backFillStartDate: 1715172779000
  bodyTemplate: ''
  changeCaptureIntervalSeconds: 180
  credentialSecretRef:
    name: fake-auth
  groupingIntervalSeconds: 15
  httpMethod: GET
  httpTimeout: 120
  internalRateLimitCount: 100
  internalRateLimitInterval: 5
  jobTemplateRef:
    apiGroup: streaming.sneaksanddata.com
    kind: StreamingJobTemplate
    name: arcane-stream-rest-api
  lookBackInterval: 86400
  pageResolverConfiguration:
    resolverPropertyKeyChain:
      - results
    resolverType: OFFSET
    responseSize: 1
    startOffset: 1
  reloadingJobTemplateRef:
    apiGroup: streaming.sneaksanddata.com
    kind: StreamingJobTemplate
    name: arcane-stream-rest-api
  responsePropertyKeyChain:
    - results
  rowsPerGroup: 10000
  sinkLocation: streaming@air-quality/openaq
  templatedFields:
    - fieldName: dateFrom
      fieldType: FILTER_DATE_BETWEEN_FROM
      formatString: yyyy-MM-ddTHH:mm:ssZ
      placement: URL
    - fieldName: dateTo
      fieldType: FILTER_DATE_BETWEEN_TO
      formatString: yyyy-MM-ddTHH:mm:ssZ
      placement: URL
    - fieldName: page
      fieldType: RESPONSE_PAGE
      formatString: ''
      placement: URL
  urlTemplate: >-
    https://api.openaq.org/v2/measurements?date_from=@dateFrom&date_to=@dateTo&limit=100&sort=desc&radius=1000&order_by=datetime&page=@page
```

```bash
$ kubectl apply -f openaq-air-quality.yaml
```

Shortly after the stream definition is created, the streaming job should be created and started.
Note that the job will start in the backfilling mode and will switch to the normal mode after the backfill is completed.
