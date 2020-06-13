---
layout: post
title:  "Introduction to Kafka Streams in Kotlin"
date:   2020-06-13 10:00:00 +0100
tags: Kotlin Kafka
excerpt: "Learn how to use Kafka Streams in Kotlin, learn about stream transformers, and look at the code step by step."
card: "summary_large_image"
image: "/assets/kafka-streams-transformer.png"
author: "casasprunes"
---
## 1. Overview

In this article, we will learn how to use Kafka Streams in Kotlin.

## 2. Creating a Kafka Stream

Kafka Streams combine the simplicity of writing and deploying Kotlin applications with the benefits of Kafka’s server-side cluster technology.   

It is a library for building applications, which stores the input and output data in Kafka clusters. In this section, we will see how we can create a Kafka Stream by using Kotlin.

### 2.1 Gradle Dependencies

First, let us add the following dependencies into our _build.gradle_:

{% highlight groovy %}
dependencies {
    implementation 'org.apache.kafka:kafka-clients:2.4.1'
    implementation 'org.apache.kafka:kafka-streams:2.4.1'
}
{% endhighlight %}

### 2.2 Building a KafkaStreams

Once we have configured our build management, let us continue by defining a stream that reads records from the _preferences-authorization_ topic,  transforms them by using the _PreferencesTransformer_ and sends the output records to either the _preferences_ topic or the _preferences-authorization_ topic.

![Kafka Streams Transformer]({{ site.url }}/assets/kafka-streams-transformer.png "Kafka Streams Transformer")

#### 2.2.1 Configuring KafkaStreams

We need to provide a set of properties for the configuration of our KafkaStreams:

{% highlight kotlin %}
val properties = Properties()
{% endhighlight %}

The identifier for the stream processing application:

{% highlight kotlin %}
properties[StreamsConfig.APPLICATION_ID_CONFIG] = config.applicationId
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

#### 2.2.2 Building a KafkaStreams Topology

We need to create a KafkaStreams topology that defines the processing logic of our stream - that is how records should be consumed, transformed and produced.

In our example, the stream consumes records from the _preferences-authorization_ topic and transforms them by using the _PreferencesTransformer_. 
Then we use branching to route records to different downstream topics based on the supplied predicates.

{% highlight kotlin %}
val builder = StreamsBuilder()

val (preferences, preferencesAuthorization) = builder
    .stream<String, SpecificRecord>(config.topics.preferencesAuthorization)
    .transform(TransformerSupplier { PreferencesTransformer(clock) })
    .branch(
        Predicate { _, v -> v is PreferencesCreated },
        Predicate { _, v -> v is CreatePreferencesDenied }
    )
{% endhighlight %}

The predicates are evaluated in order. In case of a match, the record is placed to the corresponding _Kstream_. In our example, if the record is a _PreferencesCreated_ it is placed into the _preferences KStream_ and routed to the _preferences_ topic.
 
{% highlight kotlin %}
preferences
    .peek { _, record -> logger.info("Sent ${record.schema.name} to topic: ${config.topics.preferences}\n\trecord: $record") }
    .to(config.topics.preferences)

{% endhighlight %}
 
And if the record is a _CreatePreferencesDenied_ it is placed into the _preferencesAuthorization KStream_ and routed to the _preferences-authorization_ topic.

{% highlight kotlin %}
preferencesAuthorization
    .peek { _, record -> logger.info("Sent ${record.schema.name} to topic: ${config.topics.preferencesAuthorization}\n\trecord: $record") }
    .to(config.topics.preferencesAuthorization)
{% endhighlight %}

If no predicate matches, the record is dropped.

Next, we need to build the topology:
{% highlight kotlin %}
val topology = builder.build()
{% endhighlight %}

#### 2.2.3 Starting KafkaStreams service

Finally, we need to start our KafkaStreams service by calling the _start()_ method.

{% highlight kotlin %}
val streams = KafkaStreams(topology, properties)

streams.start()
{% endhighlight %}

### 2.3 Building a KafkaStreams Transformer

We need to create the _PreferencesTransformer_ that processes the records consumed from the _preferences-authorization_ topic. For each record of type _CreatePreferencesCommand_ it needs to do a validation. If the validation is successful, it needs to return a record of type _PreferencesCreated_. If the validation is not successful, then it needs to return a record of type _CreatePreferencesDenied_. 

To keep the example simple, we added a dummy validation to check if the currency is USD, in a real application, the validations would be more complicated.

{% highlight kotlin %}
if (record is CreatePreferencesCommand) {
    val eventId = UUID.randomUUID().toString()
    val event = if (record.currency == "USD") {
        PreferencesCreated(eventId, clock.instant(), record.customerId, record.currency, record.country)
    } else {
        CreatePreferencesDenied(eventId, clock.instant(), record.customerId, record.currency, record.country)
    }
    return KeyValue(record.customerId, event)
}
{% endhighlight %}

### 3. Conclusion

In this article, we saw how to create Kafka Streams in Kotlin and how we can take advantage of the benefits of using Kafka's server-side cluster technology to consume records from Kafka topics, transform those records and route them to different downstream topics.

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/tutorials/tree/master/kafka-streams
[avro]: https://avro.apache.org/
