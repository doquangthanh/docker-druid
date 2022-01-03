# Docker Druid

Tags:

- 0.9.1, 0.9, latest ([Dockerfile](https://github.com/doquangthanh/docker-druid/blob/master/Dockerfile))

## What is Druid?

Druid is an open-source analytics data store designed for business intelligence (OLAP) queries on event data. Druid provides low latency (real-time) data ingestion, flexible data exploration, and fast data aggregation. Existing Druid deployments have scaled to trillions of events and petabytes of data. Druid is most commonly used to power user-facing analytic applications.


## Running

Druid being a complex system, the best way to get up and running with a cluster is to use the docker-compose file provided.

Clone our public repository:

```
git clone git@github.com:deitch/docker-druid.git
```

and run :

```
docker-compose up
```

The compose file will launch :

- 1 [zookeeper](https://hub.docker.com/r/znly/zookeeper/) node
- 1 postgres database

and the following druid services :

- 1 broker
- 1 overlord
- 1 middlemanager
- 1 historical
- 1 historical

The image contains the full druid distribution and use the default druid cli. If no command is provided the image will run as a broker.

If you plan to use this image on your local machine, be careful with the JVM heap spaces required by default (some services are launched with 15gb heap space).

## Configuration

You can override *any* property by setting an environment variable that begins with `property_`. The leading `property_` will be removed, the remaining `_` will be replaced by `.`, and the resultant value will be set as `-D` on the command-line.

The most common use cases are for `druid.` properties.

For example, if you want to override the setting `druid.metadata.storage.connector.user` and set it to `DBUSER`, set the environment variable `property_druid_metadata_storage_connector_user=DBUSER`. All case is respected.

Thus `property_druid_metadata_storage_connector_connectURI=jdbc:postgresql://dbSErver/db` will be converted to `druid.metadata.storage.connector.connectURI=jdbc:postgresql://dbSErver/db`

In addition, certain environment variables have special properties. If unset, they use the defaults configured in `./conf/druid/`. If set, they override the properties.

* `DRUID_XMX`: `java -Xmx${DRUID_XMX}`
* `DRUID_XMS`: `java -Xms${DRUID_XMS}`
* `DRUID_NEWSIZE`: `java -XX:NewSize=${DRUID_NEWSIZE}``
* `DRUID_MAXNEWSIZE` `java -XX:MaxNewSize=${DRUID_MAXNEWSIZE}``
* `DRUID_USE_CONTAINER_IP`: if `true`, sets `druid.host` to be the IP of `eth0` inside the container.
* `DRUID_HOSTNAME`: Convenience, equivalent to `druid_host=somename`
* `DRUID_LOGLEVEL`: Convenience, equivalent to `java -Ddruid.emitter.logging.logLevel=${DRUID_LOGLEVEL}`.

Priority overrides: lowercase settings, e.g. `druid_host`, **always** override special variables, e.g. `DRUID_HOSTNAME`.

Within the special variables, `DRUID_HOSTNAME` overrides `DRUID_USE_CONTAINER_IP`.

### S3 Configuration
Druid uses [jets3t](druid-s3-extensions) to store data segments in S3. These normally are configured by changing `jets3t.properties`. You can override those properties using the `property_` environment variables described earlier.

For example, if you want to override `s3service.s3-endpoint` to be `my.region.service.com`, set `property_s3service_s3-endpoint=my.region.service.com`.

## Env file
Although the right way to do this is via environment _variables_, some container orchestration systems don't do such a great job loading up large numbers of environment variables. As such, if the file `/druid.env` exists, its contents will be read as a `.env` file before execution of druid.

It can be done simply as `docker run -v $PWD/druid.env:/druid.env:ro ...`

## Extensions
By default, the following druid extensions are enabled:

```
["druid-histogram", "druid-datasketches", "postgresql-metadata-storage", "druid-kafka-indexing-service"]
```

If you wish to change which extensions are used, you can set the environment variable `DRUID_EXTENSIONS` as follows:

* `extension1,extension2,extension3,...,extensionN`: **replace** the existing list with the ones in the environment variable.
* `+extension1,extension2,extension3,...,extensionN`: **add** the ones in the environment variable to the ones in the default list, if the list starts with a `+` character.

Note that we do _not_ use the normal path of `druid_extensions_loadList` to override the list for 2 reasons:

1. Sometimes we want to _add_ instead of _replace_
2. The format in the config normally includes quotes and spaces and `[]` characters, which can cause a problem in a shell when set on an environment variable.

### Examples

#### Replace

1. The default list is `druid.extensions.loadList=["druid-histogram", "druid-datasketches", "postgresql-metadata-storage", "druid-kafka-indexing-service"]`.
2. Set `DRUID_EXTENSIONS=druid-my-indexer-service,druid-s3-extensions`
3. Final list is `druid.extensions.loadList=["druid-my-indexer-service", "druid-s3-extensions"]`.

#### Add

1. The default list is `druid.extensions.loadList=["druid-histogram", "druid-datasketches", "postgresql-metadata-storage", "druid-kafka-indexing-service"]`.
2. Set `DRUID_EXTENSIONS=+druid-my-indexer-service,druid-s3-extensions`
3. Final list is `druid.extensions.loadList=["druid-histogram", "druid-datasketches", "postgresql-metadata-storage", "druid-kafka-indexing-service", "druid-my-indexer-service", "druid-s3-extensions"]`.




## Logging
Some of the services are configured to output logs directly to stdout, which lets docker logs and other reasonable container logging capture services to see them. The big exception is the indexer service, run by the middlemanager. It can write only to file, S3, Azure or Google Storage, not to stdout. [see](https://github.com/druid-io/druid/blob/4ca3b7f1e43245f59737756201d037c5b7d0e8a2/docs/content/configuration/indexing-service.md).

The indexer is configured by default to write inside the container to `/var/druid/indexing-logs/`. If you need to see the logs, either exec into the container, or volume mount that directory.

## Complex Dependencies
Druid is a fairly complex product to set up. Getting it working right is not easy, and the dependencies that can cause it to break are numerous.

This section is not a complete overview of druid, but a high-level introduction to the components and, most importantly, how they coordinate. Understanding how they coordinate can help you avoid the many problems we have had getting it up and running.

Druid is composed of 5 key node types:

* overlord
* coordinator
* historical - handles analysis and searches on historical data
* real-time - handles analysis requests on real-time data and indexes data as it comes in
* broker - brokers incoming analysis requests to appropriate historical or real-time nodes

In short:

1. Data comes in
2. Data is handed to a real-time node
3. Real-time node indexes it and saves the indexed data to "deep storage"
4. Real-time node handles analysis requests as long as it is within the "real-time window"
5. Historical node uses indexed data to answer historical analysis requests

In practice, a sixth node type exists, the middlemanager. The middlemanager is responsible for creating and scaling real-time indexing workers on the fly. Thus, incoming data causes the middlemanager to create a local worker task to index the data.

The docker image is able to function in any of the 5 modes: overlord, coordinator, broker, historical, middlemanager.

### Coordination
Since these components depend upon each other and communicate, it is important to understand what they must share in order to function.

* Metadata: Metadata is stored in a single database and is accessed _only_ by coordinator nodes. In a simple setup with a single coordinator, you can use a local filesystem database, e.g. Derby. For real clusters, use real databases such as postgres. In this sample's compose, we use postgres.
* Deep storage: Deep storage is where the actual data is stored. Real-time nodex index the data and save the indexed data as segments in deep storage. Supported deep storage are: AWS S3, HDFS, local file.  Because local file must be shared across multiple nodes, it only works if it is a shared mount. In the sample compose for this project, we use file-type storage and mount a shared volume across all containers. For production, you probably should use HDFS or S3.
* Indexing logs: While creating indexes, real-time nodes record logs as files. Like deep storage, these must be accessible to multiple nodes. Supported storage are the same as deep storage: AWS S3, HDFS, local file. In the sample compose for this project, we use file-type storage and mount a shared volume across all containers. For production, you probavbly should use HDFS or S3.
* Zookeeper: Zookeeper is used to locate cluster nodes. It uses standard zookeeper protocols for connecting and communicating.

### Sensitivity
Druid is very sensitive to the existence of various directories. For example, if log or tmp directories do not exist, it will fail, often silently. We attempted to ensure that every directory necessary exists in every container. See the Dockerfile.

## Authors

- Jean-Baptiste DALIDO <jb@zen.ly>
- Clément REY <clement@zen.ly>
- Avi Deitcher <https://github.com/deitch>
