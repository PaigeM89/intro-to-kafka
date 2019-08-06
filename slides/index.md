- title : Kafka
- description : Introduction to Kafka
- author : Max Paige
- theme : night
- transition : default

***

# Kafka crash course

*** 

### What is Kafka?

Kafka is a fast, scalable, fault-tolerant, distributed, immutable log.

It is *not* an event store or a database.

A _log_ here is an append-only sequence of records, not specific to application logging.

---

Kafka is run as a cluster of _brokers_, which reference an Apache _Zookeeper_ cluster.

Brokers store _records_ in _topics_. A record is a key-value pair with a timestamp. A topic is just a named category for records.

Kafka defaults to at-least-once processing, but can also do at-most-once and exactly-once.

' zookeeper = distributed key-value store for small data sets.
'   it's robust, tested, solves some pretty hard problems
'   some brand new systems leave these problems unsolved.
' exactly once has a perf hit

---

*Kafka is good at:*

* Moving data between systems in "real time"
* Transforming or reacting to data in "real time"

("real time" meaning usually under a minute, with exceptions)

*Kafka isn't good at:*

* Persistent data rollup (like an event store can do)
* Fetching a specific record.

' you can search by offset
' talk about rollup during stream processing


---

A kafka cluster will elect one broker to be the _Controller_, which monitors the state of other brokers and communicates partition leadership to other brokers.

If the controller dies, a new controller is elected via zookeeper.

***

### Topics

A _topic_ is a feed where records are written. Topics have one or more _partitions_, where records are stored sequentially. When a topic is created, Kafka will automatically distrubute the partitions across the brokers.

![log-anatomy](images/log_anatomy.png)

---

Partitions are

* immutable
* append-only (and thus ordered)
* durable _until_ configured retention conditions are met

Each record has an _offset_ that indicates its position within a partition.

' a topic is, technically, not ordered
' retention = time, space
' horizontal scaling
' parallel consuming

---

Topics have configurable replication across brokers. 

A topic can also be configured for how many replicas must write for a producer to accept that "all" replicas have been updated.

A topic with `replication 4` and `minimum insync replicas 3` can still be written to if a broker is down.

---

Every partition has a _leader_, the broker which handles reads & writes for the partition. Other (follower) brokers replicate that leader.

If a broker dies, the partitions that it is the leader for undergo leader election. Only an in-sync replica (ISR) can be picked as the new leader.

---

### Consuming

A consumer reads records from a topic as part of a _consumer group_. Multiple consumers in the same group will have the topic's partitions split between them. Each consumer can control its offsets. Offsets are stored in the brokers at the group level.

Different consumer groups can consume from the same topic, and have no impact on each other.

' storing offsets in kafka means a consumer can leave & rejoin

---

### Producing

A producer writes data to a topic. It is responsible for choosing the partitioning strategy. Multiple producers can write to a single topic.

***

### Retention

Retention policies are configured at the topic level.

The broker can set defaults for topic retention. The default default is to retain data for 7 days.

---

There are 3 ways to configure retention:

1. Time-based, where records after X time are not guaranteed to be present.
2. Space-based, where the oldest records are pruned as long as the total size is above X.
3. Compaction, where records are matched on keys and the old versions of that record are pruned.

Retention policies can be combined

---

A compacted topic with unlimited retention will inevitably need a way to delete a record. This is done through _tombstones_.

A record with the key of the target record and a null value will cause the compaction process to delete the old record.

After some (topic-level) configured amount of time, the tombstone record itself is removed.

***

### That's Kafka's Core

Any questions so far?

***

### Schemas

Kafka is only aware of bytes. Schemas are up to you.

A topic is not schema aware. Anything can be written to the key or value in a record. This can cause problems for consumers.

_Options:_

* JSON strings
* Your favorite serialization format

---

### JSON Strings

`"{\"title\":\"JSON\",\"content"\:\"this obviously has no downsides\"}"`

* We all know the common pitfalls
* Not space efficient
* Not friendly for consumers

' multiple consumers complicates schema updates
' overy harsh casting tools can break on trivial changes, like what happened to me twice

---

### Avro

Kafka's "default" serialization format is Apache Avro - most libraries will have tools for it. 

Avro provides strict rules around schema version compatibility. Adding, changing, or removing a field can be validated to ensure forward, backward, or bi-directional compatibility.

Using avro can make sure any record written to a topic can be read by a consumer. A record must include the schema used to write it.

' A record including the schema is ALSO not space efficient; in fact, can be less so

---

### Schema Registry

Confluent's open-source tooling includes the _schema registry_, an application which stores Avro schemas accessible in an API. 

* Can automatically check compatibility when registering a schema
* Allows a record to include the _id_ of the schema used to write it.

' massive gains in space efficiency! hooray!

***

### Connect

Confluent also offers _Kafka Connect_, a pluggable API to source or sink data from a variety of systems.

---

Connect allows you to create or alter a _connector config_ to stream data from a system.

A _connector plugin_ is the class used for interfacing with a data system. Lots already exist, and custom ones can also be plugged into Connect.

' plugin examples: sql server, postgres, twitter, blockchain

---

Example Connector Config:

```
{
    "connector.class" : "io.confluent.connect.cdc.mssql.MsSqlSourceConnector",
    "tasks.max" : "1",
    "initial.database" : "testing",
    "username" : "user",
    "password" : "secret",
    "server.name" : "db-01.example.com",
    "server.port" : "1433",
    "table.whitelist":"MyTable",
    "topic.prefix":"sqlServer-",
    "transforms":"setKey",
    "transforms.setKey.type":"org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.setKey.fields":"Id"
}
```

---

Connect easily interfaces with the schema registry for schema creation or validation.

A connector config can have _transforms_ specified to alter records as they are processed; these are usually very simple, but transforms are pluggable.

***

### Stream Processing

Kafka Streams is really just a fancy consumer/producer wrapper API. Stream processors are regular JVM apps, without any special requirements.

Because they're built on consumers, they get all the benefits of consumer groups - scalable, dynamic rebalancing, etc.

---

The streams API is built around one-record-at-a-time semantics (but it's actually microbatches of around 500 under the hood (you'll never need to know this)).

```
val builder = new StreamsBuilder

builder.stream[String, MyValue]("sqlServer-MyTable")
  .mapValues(x => new OtherValue(x))
  .to("my-other-topic")
```

' the api builds a graph that will auto-created internal topics as needed.
' this example is stateless

---

The API has features for stateful processing, which is required for joining or aggregating streams.

A _stream_ is the feed of data off a topic, and is viewed as an unending flow of data. It can also be viewed as a changelog for a table. Incoming data on a stream can be merged into a table to represent current state.

If there are multiple stream processor instances (thus multiple consumers), the resulting table will be split across those instances.

' state is also required for rollup

---

All joins require some kind of state. Joins may require one or both streams to be re-keyed so that they are processed by the same consumer.

Tables can be made _global_, where the entire table is loaded on every instance of a stream processor.

' global tabling is great for small tables, bad idea for more than a few million records.

**Stream -> Table** joins are table lookups on keys. A stream joining to a global table can use values for lookups.

**Stream -> Stream** joins require some kind of _Windowing_ to limit how far back data is retained.

---

Windowing has 4 options:

1. Tumbling windows: Fixed-size, non-overlapping, no gaps.
2. Hopping windows: Fixed-size, overlapping. 
3. Sliding windows: Fixed-size, overlapping windows that are built on timestamp differences.
4. Session windows: Dynamically-sized, non-overlapping, data-built windows.

' tumbling, hopping, sliding: PER RECORD KEY

---

![streams-stateful_operations](images/streams-stateful_operations.png)

***

### In the wild

*Monitoring & Alerting*

Kafka exposes a _lot_ of metrics to monitor. Some of the most important ones are:

1. The amount of replicas not in-sync. If this stays above 0 for a short length of time (more than a minute), there's an issue.
2. The amount of controllers in the cluster. If it's more than 1, it's Very Bad (TM).
3. The amount of brokers currently in the cluster. If it's lower than the amount of brokers, there's an issue.

---

Stream processors should monitor:

1. Consumer lag. If a consumer's lag starts to climb and the offset isn't incrementing, it's not processing records.
2. Application status (such as through systemd).

Other fields, like bytes/sec, are great for checking health, but aren't good for alerts due to fluctations in data rates.

Kafka Connector statuses need to be polled through the API.

Kafka & Friends produce a _huge_ amount of logs and all of them (info+) should be collected.

---

*Other thoughts*

Confluent recommends using whatever deployment the organization is most comfortable with (physicals, VMs, containers).

Disk IO has a _huge_ impact on performance. Excess RAM on the brokers for paging goes a long way. 
' if using spinny drives, RAID config can really matter.

Kafka is _fast_. It can easily support 50,000 to 100,000 writes per second.

' we once overloaded our prod firewall without kafka breaking a sweat.

---

If a broker has a lot of partitions, the process will need a _lot_ of file handlers - possibly unlimited.

' if you're too low on file handlers, the system will crash on startup with a useless error message, and it can be very difficult to debug.

Kafka is a distributed system. Joining records may require custom-implemented retry logic if records arrive out of order. At-least once messaging means downstream systems need to be able to handle duplicates.

' duplicate handling can end up being sort of expensive depending on the shape of your data. 

---

A "typical" kafka cluster will have:

* 3 or 5 Zookeepers
* 3 or more Kafka brokers
* 1 or more Schema Registries
* 1 or more Connect instances
* Any amount of stream processors

Building, configuring, and maintaining these can be a lot of work. Cloud options may limit configurability.

*** 

**References**

[The Log - Jay Krepps](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)

[Kafka docs](https://kafka.apache.org/documentation/)

