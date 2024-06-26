# Arcane REST API streaming plugin

## Overview

This plugin is designed with the capability to transfer data from multiple REST API sources.
It streams this data to a storage system that is compatible with Amazon S3. The data is formatted as JSON files for
compatibility and ease of use. In these JSON files, each piece of data or 'data object' is converted into a
'JSON-serialized entry'. These entries are separated by line breaks, making each one distinct and easier to handle
individually.


## Quick Start guide

You can see an installation example in the [quick start guide](../arcane-rest-api-main/docs/quickstart.html).

## Configuration guide
The plugin can be installed via Helm Chart, configuration is set via
[values.yaml](https://github.com/SneaksAndData/arcane-stream-rest-api/blob/main/.helm/values.yaml)

The short configuration options descriptions also available in the
[configuration guide](../arcane-rest-api-main/docs/configuration.html).


## Source code and packages

The source code is available in the [GitHub repository](https://github.com/SneaksAndData/arcane-stream-rest-api).
The docker image can be pulled from [GitHub container registry](https://github.com/SneaksAndData/arcane-stream-rest-api/pkgs/container/arcane-stream-rest-api).
The Helm Chart can be pulled from [GitHub container registry](https://github.com/SneaksAndData/arcane-stream-rest-api/pkgs/container/helm%2Farcane-stream-rest-api)
