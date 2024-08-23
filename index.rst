:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::


Abstract
=========

We describe the proposed design and implementation of the LSST Alert Distribution System, which provides rapid dissemination of alerts to community alert brokers.
At time of writing, this service is still under development; this “living document” describes current thinking, but is expected to evolve over the course of LSST construction.

Alert Serialization
===================

Packet Format
-------------

Alerts are packaged using Apache Avro :cite:`avro`.
Avro is a framework for data serialization in a compact binary format.
It has been used at scale in both industry and science, and it is the recommended format for data streamed with Apache Kafka.
Avro is more structured in format than JSON or XML, the currently used format of VOEvent 2.0.
Furthermore, image — or other — files can be embedded in Avro packets, making it possible to embed postage stamp cutouts of detected difference image sources in a much more compact and convenient way than the current VOEvent standard.
Libraries for reading and writing Avro are available in many languages, including Python.

Each alert is packaged as its own Avro packet, as opposed to wrapping groups of alerts per visit together.
Packaging alerts separately allows brokers to take individual alerts as input and process each alert independently without having to repackage groups.
As alerts are anticipated to arrive independently from the end of the Alert Generation Pipeline parallelized by CCD, the Kafka platform (see :ref:`alertDist`) acts as a cache before distribution, and individually packaging alerts makes this process simple.

Alert Schemas
-------------

Purpose and Structure
^^^^^^^^^^^^^^^^^^^^^

Avro alert packets are composed of opaque binary data.
Interpretation of this data is done according to a *schema*, which defines the type and structure of each field within the packet.
Strict adherence to the schema ensures that data will be correctly interpreted upon receipt.

Avro schemas can be composed of nested sub-schemas under a top level namespace.
Nesting simplifies what would otherwise be monolithic schemas as new fields are added.
For example, the base alert schema (``lsst.alert``) is of type "record" and includes previous detections of DIA sources as an array of type ``lsst.alert.diaSource``.

The current schemas contain all fields specified by the LSST Data Products Definition Document (:cite:`LSE-163`).
At this stage in construction, this schema should be regarded as exploratory and subject to rapid change; as we move closer to the operational era, a change control process will be implemented.

The current schema proposed for use with LSST alerts is stored in the `lsst/alert_packet`_ repository.
This primary purpose of this package is to enable generation of alerts during Prompt Processing.
Versioning of this package with the rest of the Science Pipelines software ensures reproducibility.
During operations, users will access the production schemas using other tools (see :ref:`alertManagement`).

.. _lsst/alert_packet: https://github.com/lsst/alert_packet

.. _alertManagement:

Management and Evolution
^^^^^^^^^^^^^^^^^^^^^^^^

While a schema is required to interpret Avro's binary data, to avoid duplication many industrial users of Kafka ship Avro payloads without including the schema.
Since the schema can be lengthy, and since each alert will be packaged for shipping separately, we expect to distribute LSST alerts to community alert brokers without including the associated schema.

The recipient of each packet requires a schema to interpret the data they have been sent.
In the simplest model, the producer (the LSST Alert Distribution System) and the consumer (the remote broker) simply agree on the schema prior to the start of the stream.
In practice, however, this is impractical: over the course of LSST operations, we anticipate that the alert schema will evolve in response to changing technical and scientific requirements.
At the same time, this change must be managed so as to cause minimum disruption to consumers.

Given the concerns above, we expect to adopt the following protocol:

- The LSST Project will make available a registry of all alert schemas ever used operationally by LSST.
  Schemas may be retrieved from this registry by some convenient interface given a four-byte schema ID.
  We plan to use the `Confluent Schema Registry`_, although it is possible an alternative implementation will be deployed in practice.
- Alerts will be transmitted following the `Confluent Wire Format`_.
  That is, the alert data encoded in Avro format will be prepended with a “magic byte” indicating the version of the wire format in use and the four-byte schema ID.
- On receipt of an alert packet, an alert broker can retrieve the appropriate schema from the registry before attempting to interpret the packet.
  (Consumers are expected to cache the schema, rather than requesting a fresh copy of it for every packet received!)

LSST alert schemas will follow a ``MAJOR.MINOR`` versioning scheme.

Within a given ``MAJOR`` version, schemas will follow the ``FORWARD_TRANSITIVE`` type of `Confluent compatibility model`_.
In this model, data produced by a newer schema can be interpreted by a consumer using an older schema.
The producer may add fields to the schema (which will not be seen by the consumer) and may delete *optional* fields (in which case the consumer will see the default value).
In this way, LSST may add to or augment the contents of alert packets without impacting consumers (of course, consumers who wish to take advantage of the new information available will have to upgrade their systems to match the new schema).
All such additions or augmentations to the schema will result in a new ``MINOR`` version being generated.

In some cases, a break with compatibility may be required (for example, when some particular data product is rendered obsolete, deleting the corresponding field from the alert schema will break the ``FORWARD_TRANSITIVE`` compatibility guarantee).
Such a break with compatibility will be signified by the release of a new ``MAJOR`` version of the schema.
Issuing a new major schema will require that consumers not using the schema registry to manually update their kafka
deserializer to use the newest schema.

The Confluent Schema Registry makes it possible for schemas to evolve within a given *subject name*. New schemas
added to a given subject name will have a unique id and version number. To maintain a consistent registry history,
the ``FORWARD_TRANSITIVE`` compatibility is not enforced within the schema. All major and minor
schema changes are instead given a unique schema id based on the major and minor version defined in `lsst/alert_packet`_.
For example, schema version 7.1 is given schema id 701. For schema version 13.12, we would have schema id 1312. This allows us to maintain
consistency so all historical alerts can be read even if the schema registry has to be recreated.

Archival disk space is somewhat less constrained than the outbound bandwidth required for the real-time alert stream.
To ensure that alerts can be read independently of the Project's Schema Registry, all alerts that we store on disk will include the schema they were written with. 
For convenience and efficiency we will frequently store many alerts together in single Avro files sharing a single schema.
Users can then read these files directly with existing Avro libraries.

.. _Confluent Schema Registry: https://docs.confluent.io/current/schema-registry/docs/index.html
.. _Confluent Wire Format: https://docs.confluent.io/current/schema-registry/docs/serializer-formatter.html#wire-format
.. _Confluent compatibility model: https://docs.confluent.io/current/schema-registry/docs/avro.html#forward-compatibility

Example Data
^^^^^^^^^^^^

At present, Avro files populated with precursor data following the published schema are available at locations specified in the `lsst-dm/sample_alert_info`_ repository.
Although we expect to continue to make example alert data available for the indefinite future, the contents, format, and location is subject to change with time.

.. _lsst-dm/sample_alert_info: https://github.com/lsst-dm/sample_alert_info/

.. _alertDist:

Alert Distribution
==================

Alert distribution uses Apache Kafka :cite:`kafka`,
an open source streaming platform
that can be used for real-time and continuous data pipelines.
Kafka is a scalable pub/sub message queue based on a commit log.
It is used in production at scale at companies such as LinkedIn,
Netflix, and Microsoft to process over 1 trillion messages per day.

Kafka collects messages from processes called "producers,"
which are organized into distinct streams called "topics."
Downstream "consumers" pull messages by subscribing to topics.
Topics can be split into "partitions" that may be distributed
across multiple machines and allow consumers to read in
parallel as "consumer groups."
Data can be replicated by deploying Kafka in cluster mode over several
servers called "brokers."
We will refer to these brokers below as "Kafka brokers" to distinguish
from the LSST alert downstream "community brokers" that will process
LSST alerts.

For LSST alert distribution, Kafka and the accompanying Zookeeper
can be deployed as Docker containers from the DockerHub image repository
maintained by Confluent Inc., the team that created Kafka.
The latest release of ``alert_stream`` uses Kafka and Zookeeper from
Confluent platform release 4.1.1, which was the latest version available
as of the dmtn-081-2018-06-18 tagged release of ``alert_stream``
used in :cite:`DMTN-081`.
As of the writing of this document, Confluent platform release 6.1
corresponding to Apache Kafka version 2.7 is now available.
The producer used for generating and sending data to Kafka and
template scripts for consumers of the stream are provided in the GitHub
repository at https://github.com/lsst-dm/alert_stream,
which can also be built as a Docker image and deployed as containers.
:cite:`DMTN-028`
provides details about benchmarking deployment of the different components.

Alert Filtering
================

Selected community alert brokers will receive the full LSST alert stream and provide a range of user tools to identify alerts of interest.
We are currently evaluating technical approaches for LSST-hosted filtering of the alert stream for users with LSST Data Rights (see :cite:`RDO-013`).
:cite:`DMTN-165` presents one potential option of a "hybrid" system that provides users a lightweight stream containing summaries of *all* alerts. 
Users of the hybrid service could then retrieve the full-sized alerts corresponding to the subset of events of interest from the Alert Database.

Alert Database
==============

The Alert Database provides an archival record of alerts sent to community alert brokers.
Users with LSST Data Rights can access the Project-hosted service to retrieve alerts of interest.
:cite:`DMTN-183` describes the technical design envisioned for the Alert Database.

Deployment
===========

Deployment scripts for deploying a full mini-broker configuration
(a producer, central Kafka instance, filtering Kafka instances,
filters, and consumers) are available in the `lsst_dm/alert_stream`_ repo.
These scripts are specifically for a deployment using Docker Swarm or Kubernetes.
Complete instructions for deploying on an AWS CloudFormation cluster
are included with the deployment scripts in the swarm directory
of alert_stream.


.. _lsst-dm/alert_stream: https://github.com/lsst-dm/alert_stream

Remaining Work
===============

There is remaining work particularly in addressing questions around
resilience, how users interface with the system, and
feasibility of some "desirements."
Below are a few (non-exhaustive) outstanding questions and thoughts.

* How can we make the system resilient to a node going down?

It is probable that we will use Kafka in cluster mode and
take advantage of consumer groups.

* How do we back up alerts?

Containers running Kafka should not use local storage (inside the
container) to store alerts but should use volume mounted disk.
Storage should be mounted to the /var/lib/kafka/data directory
inside the container.
If using Kafka in cluster mode, replication to > 1 can be set.
The volume mounted disk should also be backed up for as long as
data needs to be kept accessible via Kafka.

* How should we organize streams/topics?

It makes sense to create a new topic on a daily basis to make
it straightforward for downstream consumers to listen to
a night's worth of data, separate data of interest, and not
overwhelm consumers who want to, e.g., replay a night from last
week without reprocessing all alerts available since then.
Daily topics also make expiring nights of data straightforward
instead of ending up expiring data somewhere in the middle
of the night.
However, daily topics require more manual management by downstream consumers, and large numbers of Kafka topics can create stability issues. 
Further investigation and discussion with community alert brokers are warranted.

* For how long should we persist streams?

This is also partially a policy question.
The default setting in Kafka is to persist data for one week,
so topics older than one week could be removed.
(The topics will still exist unless deleted, but they will contain no alerts.)
Expiration of data can be set by a time limit or a storage cap.
The amount of time we will cache / allow “rewindable” access to the alert
stream and the number of partitions configured for each topic
sets requirements on the sizes and number of disks needed for storage.
See :cite:`DMTN-028` for compute resource recommendations for different scenarios.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
