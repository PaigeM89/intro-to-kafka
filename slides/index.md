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
' exactly once has a perf hit (10% - 20%?)

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

Brokers may need a lot of file handlers on startup (possibly unlimited).

---

### Consuming

A consumer reads records from a topic as part of a _consumer group_. Multiple consumers in the same group will have the topic's partitions split between them. Each consumer can control its offsets. Offsets are stored in the brokers at the group level.

Different consumer groups can consume from the same topic, and have no impact on each other.

' storing offsets in kafka means a consumer can leave & rejoin

---

![consumer-groups](images/consumer-groups.png)

---

### Producing

A producer writes data to a topic. It is responsible for choosing the partitioning strategy. Multiple producers can write to a single topic.

***

### Retention

Retention policies are configured at the topic level.

The broker can set defaults for topic retention. The default default is to retain data for 7 days.

' kafka's retention, replication, distribution make it work well as a storage system

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

Connect easily interfaces with the schema registry for schema creation or validation.

A connector config can have _transforms_ specified to alter records as they are processed; these are usually very simple, but transforms are pluggable.

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

Some actions will sink data back to the cluster in their own topic.

```
builder.stream[...]
  .selectKey(x => x.Id)
```

This will sink data to an auto-generated topic, which allows re-keyed records to be partitioned differently so that the correct stream processor instance will see those records. 

' sinking data is relevant for joins and for fault tolerancd

---

The API has features for stateful processing, which is required for joining or aggregating streams.

A _stream_ is the feed of data off a topic, and is viewed as an unending flow of data. It can also be viewed as a changelog for a table. Incoming data on a stream can be merged into a table to represent current state.

If there are multiple stream processor instances (thus multiple consumers), the resulting table will be split across those instances.

' state is also required for rollup

---

![streams-architecture-state](images/streams-architecture-states.jpg)

' multiple stream processor instances, input splitting, output sinking

---

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

When using something with Windowing, you may want to change how timestamps are selected on a record.

* Record timestamp (default timestamp extractor)
* Wallclock timestamp (timestamp on the stream processor instance)
* A custom timestamp extractor that retrieves a field from a record

```
val timestampExtractor = new TimestampExtractor {
  def extract(record: ConsumerRecord[MyKey, MyValue], previousTimestamp: Long) = ???
}
val consumed = Consumed.`with`(keySerde, valueSerde, timestampExtractor)
```

' using a different timestamp extractor changes the _record timestamp_ when sunk, which is usually helpful
' serde - serializer / deserializer, using Avro. Avro support is built into streams

---

![streams-stateful_operations](images/streams-stateful_operations.png)

' relationships & transforms between stream processor types 

--- 

Kafka streams brings two main problems in distributed systems: 

* Out of order messages
* Duplicate messages

These require careful consideration & planning. Downstream systems may need to be more resilient to duplicate messages. Retry loops may be needed on failed joins.

Broken records may require DLQ or other error handling processes.

---

An example stream processor that counts words from input text:

```
  val builder = new StreamsBuilder()

  val textLines: KStream[String, String] = 
    builder.stream[String, String]("streams-plaintext-input")

  val wordCounts: KTable[String, Long] = textLines
    .flatMapValues(textLine => 
        textLine.toLowerCase.split("\\W+"))
    .groupBy((_, word) => word)
    .count()

  wordCounts.toStream.to("streams-wordcount-output")
```

***

A "typical" kafka cluster will have:

* 3 or 5 Zookeepers
* 3 or more Kafka brokers
* 1 or more Schema Registries
* 1 or more Connect instances
* Any amount of stream processors, often with multiple instances of each

Building, configuring, and maintaining these can be a lot of work. Cloud options may limit configurability.

' confluent says to use whatever the org is comfortable with to run kafka

*** 

**References & Additional Reading**

[Kafka docs](https://kafka.apache.org/documentation/)

[The Log - Jay Krepps](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)

[Desiging Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321/ref=sr_1_2?gclid=EAIaIQobChMIgcGazLza4wIVgYbACh0hhAA5EAAYASAAEgIzF_D_BwE&hvadid=243358978238&hvdev=c&hvlocphy=9015306&hvnetw=g&hvpos=1t1&hvqmt=e&hvrand=3459470570870667457&hvtargid=kwd-408354251024&hydadcr=16437_9739787&keywords=designing+data+intensive+application&qid=1564415157)

' "The Log" is inspiration / design thought behind kafka
' DDIA barely touches kafka but covers every other aspect of what kafka feeds / does / helps solve. good reading.