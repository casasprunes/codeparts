---
layout: post
title:  "Create a Read Model using Kafka Streams State Stores"
date:   2020-03-08 20:00:00 +0100
modified: 2020-06-21 10:00:00 +0100
last_modified_at: 2020-06-21 10:00:00 +0100
tags: Kotlin Kafka Ratpack
excerpt: "A Read Model built in this way could safely be discarded and rebuilt from scratch whenever it has to change, due to bugs or new requirements."
card: "summary_large_image"
image: "/assets/kafka-streams-state-store.png"
author: "casasprunes"
---
## 1. Overview

In this article, we will learn how to create a Read Model using Kafka Streams State Stores.

## 2. Persistent State Stores

Stream processing applications can use persistent State Stores to store and query data; by default, Kafka uses [RocksDB][rocksdb] as its default key-value store. 

![Kafka Streams State Store]({{ site.url }}/assets/kafka-streams-state-store.png "Kafka Streams State Store")

### 2.1 Changelog Kafka Topic

Kafka provides fault tolerance and automatic recovery for persistent State Stores; for each store, it maintains a replicated changelog topic to track any state changes. 

![State Store Stores Changelog Topic]({{ site.url }}/assets/state-store-stores-changelog-topic.png "State Store Stores Changelog Topic")

These changelog topics have dedicated partitions for each State Store. In case of a failure and restart, Kafka guarantees to restore State Stores to the content before the failure by replaying the corresponding changelog topics.

![State Store Restores Changelog Topic]({{ site.url }}/assets/state-store-restores-changelog-topic.png "State Store Restores Changelog Topic")

The changelog topics have log compaction enabled to delete old data safely to prevent the topics from growing indefinitely.

### 2.2 Distributed State

The full state of an application is usually split across multiple distributed instances, and across multiple State Stores that are managed locally by these application instances.

![Kafka Distributed State]({{ site.url }}/assets/kafka-distributed-state.png "Kafka Distributed State")

The diagram above is an example of an application that stores the balance of a customer on a State Store, every time we add funds; the application needs to read the current balance from the State Store, sum the funds and save the resulting sum back into the State Store. 

Since there are two topic partitions, two application instances and two State Store partitions, Kafka distributes events across the two topic partitions, and the applications persist data on two State Store partitions.

### 2.3 Interactive Queries

Kafka Streams allows direct read-only queries of the State Stores by applications external to the streams application that created the State Stores, through a feature called Interactive Queries.

For example, in the following diagram, we can see how we can get the balance of a customer via an Http call.

![Kafka Interactive Queries Local State Store]({{ site.url }}/assets/kafka-interactive-queries-local-state-store.png "Kafka Interactive Queries Local State Store")

In this case, the data was present in the State Store assigned to the streams application instance. If that was not the case, we should discover the remote instance that contains the data and proxy the Http call to that instance as we can see in the following diagram.

![Kafka Interactive Queries Remote State Store]({{ site.url }}/assets/kafka-interactive-queries-remote-state-store.png "Kafka Interactive Queries Local Remote Store")

With interactive queries, we can make the full state of our application queryable by:

1. Adding a Web API so that we can call instances of our application via Http
2. Exposing the Http endpoints of our application instances via the _application.server_ configuration setting of Kafka Streams
3. Discovering remote application instances and their State Stores and forward queries to other app instances if a particular instance lacks the local data to respond to a query

## 3. Creating a Read Model

A read model is a model optimized for queries. For example, in a read model optimized for querying a customer's balance, we would store the balance in the read model, and every time we consume _FundsAdded_ events, we would update it and store the result.

In this section, we will see how we can do this by using Kafka Streams State Stores.

### 3.1 Gradle Dependencies

First, let us add the following dependencies into our _build.gradle_:

{% highlight groovy %}
dependencies {
    implementation 'org.apache.kafka:kafka-clients:2.4.1'
    implementation 'org.apache.kafka:kafka-streams:2.4.1'
}
{% endhighlight %}

### 3.2 Building a KafkaStreams

Once we have configured our build management, let us continue by defining a [kafka stream][kafka-streams] that reads events from the _balance_ topic and process them by using the _BalanceProcessor_ and the _balanceReadModel_ State Store.

#### 3.2.1 Configuring KafkaStreams

We need to provide a set of properties for the configuration of our KafkaStreams:

{% highlight kotlin %}
val properties = Properties()
{% endhighlight %}

The identifier for the stream processing application:

{% highlight kotlin %}
properties[StreamsConfig.APPLICATION_ID_CONFIG] = config.applicationId
{% endhighlight %}

A host:port pair used for discovering the locations of state stores:

{% highlight kotlin %}
properties[StreamsConfig.APPLICATION_SERVER_CONFIG] = "${hostInfo.host()}:${hostInfo.port()}"
{% endhighlight %}

Default serializer/deserializer classes for keys and values:

{% highlight kotlin %}
properties[StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG] = Serdes.String()::class.java
properties[StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG] = SpecificAvroSerde::class.java
{% endhighlight %}

URLs to our local Kafka cluster and Schema Registry and the strategy on how to construct the subject name under which the value schema is registered with the schema registry. By setting _TopicRecordNameStrategy_, it allows us to publish different types of [AVRO][avro] to the same topic:

{% highlight kotlin %}
properties[StreamsConfig.BOOTSTRAP_SERVERS_CONFIG] = config.bootstrapServersConfig
properties[KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG] = config.schemaRegistryUrlConfig
properties["value.subject.name.strategy"] = TopicRecordNameStrategy::class.java
{% endhighlight %}

#### 3.2.2 Building a KafkaStreams Topology

We need to create a KafkaStreams topology that defines the processing logic of our stream - that is how events should be consumed and processed.

In our example, the stream consumes events from the _balance_ topic and process them by using the _BalanceProcessor_. The _balanceReadModel_ State Store has to be added to the topology as well, and we need to pass the State Store name to the _process_ function.

{% highlight kotlin %}
val builder = StreamsBuilder()

builder
    .addBalanceStateStore(config)
    .stream<String, SpecificRecord>(config.topics.balance)
    .process(ProcessorSupplier { BalanceProcessor(config) }, config.stateStores.balanceReadModel)

val topology = builder.build()
{% endhighlight %}

For adding the State Store to the topology, we can create an extension function on the _StreamsBuilder_ to add the _balanceReadModel_ State Store as a persistent key-value store with a _String_ as the key and a _BalanceState_ [AVRO][avro] as the value.

{% highlight kotlin %}
private fun StreamsBuilder.addBalanceStateStore(config: KafkaConfig): StreamsBuilder =
    addStateStore(
        Stores.keyValueStoreBuilder(
            Stores.persistentKeyValueStore(config.stateStores.balanceReadModel),
            Serdes.String(),
            SpecificAvroSerde<BalanceState>().apply {
                configure(mapOf(SCHEMA_REGISTRY_URL_CONFIG to config.schemaRegistryUrlConfig), false)
            }
        )
    )
{% endhighlight %}

#### 3.2.3 Starting KafkaStreams service

Finally, we need to start our KafkaStreams service by calling the _start()_ method.

{% highlight kotlin %}
val streams = KafkaStreams(topology, properties)

streams.start()
{% endhighlight %}

### 3.3 Building a KafkaStreams Processor

We need to create the _BalanceProcessor_ that processes the events consumed from the _balance_ topic. When processing events of type _FundsAdded_, it needs to update the balance in the State Store.

{% highlight kotlin %}
override fun process(key: String, record: SpecificRecord) {
    if (record is FundsAdded) {
        val newBalance = (stateStore.get(record.customerId)?.amount ?: BigDecimal.ZERO) + record.amount
        stateStore.put(record.customerId, BalanceState(record.customerId, newBalance))

        logger.info("Processed ${record.schema.name}\n\trecord: $record")
    }
}
{% endhighlight %}

### 3.4 Handling Http requests

In our Web API, we need an endpoint to get the balance of a customer. When receiving a request for a customer id, we need to discover remote application instances and the State Stores they manage locally to find the instance that holds the data for the given key in the State Store.

{% highlight kotlin %}
val customerId = ctx.request.queryParams["customerId"]
val metadata = streams.metadataForKey(config.stateStores.balanceReadModel, customerId, StringSerializer())
val hostInfo = metadata.hostInfo()
{% endhighlight %}

Once we have the host info, we need to check if it is the same instance that is handling the Http request and if that is the case, we need to query the local State Store to find the customer's Balance.

{% highlight kotlin %}
if (hostInfo == currentHostInfo) {
    logger.info("Reading local state store on ${hostInfo.toUrl()}")

    val store = streams.store(
        config.stateStores.balanceReadModel,
        QueryableStoreTypes.keyValueStore<String, BalanceState>()
    )

    ctx.response.status(Status.OK)
    ctx.render(Jackson.json(BalancePayload(store.get(customerId).amount)))
}
{% endhighlight %}

If it is not the same instance, then we need to forward the Http call to the application instance that holds the data. There is no guarantee that the data will be there, Kafka only guarantees that for a given key if there were any data stored, it would be there, but the key may not exist in the State Store. 

{% highlight kotlin %}
logger.info("Proxy to remote state store on ${hostInfo.toUrl()}")

val httpClient: HttpClient = ctx.get(HttpClient::class.java)
val uri = URI.create("http://${hostInfo.toUrl()}/api/customers.getBalance?customerId=$customerId")

httpClient.get(uri).then { response ->
    ctx.response.status(Status.OK)
    ctx.render(response.body.text)
}
{% endhighlight %}

### 4. Conclusion

In this article, we saw how to create a Read Model using [Kafka Streams][kafka-streams] State Stores and how Kafka natively provides all of the required functionality for interactively querying the state of our application, except to allow application instances to communicate over Http for which we must create a Web API.

A Read Model built in this way could safely be discarded and rebuilt from scratch whenever it has to change, due to bugs or new requirements. 
As a consequence of our _balance_ topic being treated as an Event Store and having infinite retention, we could create a new read model at any time, and it would retroactively include everything from the past.

The full source code is available [on GitHub][github].

### References:

* [Streams Architecture State][state-stores]
* [Kafka Streams Interactive Queries][interactive-queries]

[github]: https://github.com/casasprunes/tutorials/tree/master/kafka-interactive-queries
[rocksdb]: https://rocksdb.org/
[avro]: https://avro.apache.org/
[kafka-streams]: /2020/06/13/kafka-streams-kotlin/
[interactive-queries]: https://docs.confluent.io/current/streams/developer-guide/interactive-queries.html
[state-stores]: https://docs.confluent.io/current/streams/architecture.html#streams-architecture-state

