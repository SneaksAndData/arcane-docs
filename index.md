---

_layout: landing

---

# **ARCANE**


* Arcane is a data streaming platform based on Akka.NET that focuses on providing a simple, reliable,
and scalable solution for data streaming. It is designed to be Kubernetes-native and is built to be cloud-agnostic.

* Unlike other data streaming solutions like Kafka, Arcane is not based on any kind of consensus algorithm.
Instead, it uses a simple and reliable approach to data streaming that is based on the principles of the Actor Model.

* Arcane is implemented as a Kubernetes Operator that manages independent streaming applications isolated in the 
Kubernetes jobs.

Arcane utilizes a plugin architecture that allows extending its functionality with
custom source and sink plugins.

![Arcane overview](images/overview.jpg){width=1076 height=1076,  style="display: block; margin-left: 15%"}

[Concepts](concepts.md) overview.

[Quickstart](quickstart.md) deployment guide.

[Plugins](plugins.md) development guide.

[Operator Architecture](architecture.md) overview.

&nbsp;

# Stream plugins

## Currently supported stream plugins in production-ready state:
- [REST API](rest_api/overview.md)

## Stream plugins coming soon:
- [Microsoft SQL Server](https://github.com/SneaksAndData/arcane-stream-sqlserver-change-tracking/issues/7)
- [Microsoft Dynamics 365 Change Feed](https://github.com/SneaksAndData/arcane-stream-cdm-change-feed/issues/7)
- [Salesforce Bulk API](https://github.com/SneaksAndData/arcane-stream-salesforce/issues/5)

## Public roadmap
Arcane is under active development. Check the [roadmap](https://github.com/orgs/SneaksAndData/projects/21) for the upcoming features.
