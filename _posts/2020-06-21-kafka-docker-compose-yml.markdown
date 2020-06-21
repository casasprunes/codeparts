---
layout: post
title:  "Run Apache Kafka with Docker Compose"
date:   2020-06-21 10:00:00 +0100
tags: Kafka DevOps
excerpt: "Learn how to run Apache Kafka locally using Docker Compose with this example of a docker-compose.yml file."
author: "casasprunes"
---
## 1. Overview

In this article, we will learn how to run Kafka locally using Docker Compose. 

## 2. Creating a docker-compose.yml file

First, let us create a file called _docker-compose.yml_ in our project directory with the following:

{% highlight yml %}
version: "3.8"
services:
{% endhighlight %}

This compose file will define three services: _zookeeper_, _broker_ and _schema-registry_.

## 2.1. Defining Zookeeper Configuration

We need to define the _zookeeper_ service configuration. 

Apache Kafka uses Zookeeper to store its metadata. It is necessary to run a Kafka cluster; it stores data such as the configuration of topics, the number of partitions per topic, the location of partitions, etc.

{% highlight yml %}
  zookeeper:
    image: confluentinc/cp-zookeeper:5.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
{% endhighlight %}

In our example, we use the following ZooKeeper settings:
* `ZOOKEEPER_CLIENT_PORT`: It tells ZooKeeper where to listen for client connections
* `ZOOKEEPER_TICK_TIME`: The length of a single tick in milliseconds, it is used for heartbeats and timeouts

## 2.2 Defining Kafka Broker Configuration

We need to define the _broker_ service configuration. It depends on the _zookeeper_ service for stored Kafka metadata.

{% highlight yml %}
  broker:
    image: confluentinc/cp-server:5.5.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
{% endhighlight %}

In our example, we use the following Kafka settings:
* `KAFKA_BROKER_ID`: The broker id for this node
* `KAFKA_ZOOKEEPER_CONNECT`: The host and port of the ZooKeeper server
* `KAFKA_ADVERTISED_LISTENERS`: The listeners to publish to ZooKeeper for clients to use
* `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP`: The map between listener names and security protocols
* `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR`: The replication factor for the offsets topic
* `KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS`: The amount of time the group coordinator will wait for more consumers to join a new group before performing the first rebalance.
* `KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR`: The replication factor for the license topic
* `KAFKA_AUTO_CREATE_TOPICS_ENABLE`: Enable or disable auto-creation of topic on the server

## 2.3. Defining Schema Registry Configuration

We need to define the _schema-registry_ service configuration. 

Apache Kafka uses Schema Registry to store a versioned history of all schemas. Producers write data to topics, and consumers read data from topics, there is a contract that producers write data with a schema that consumers can read, even when the schemas evolve. Schema Registry does compatibility checks to ensure that the contract is not broken.

{% highlight yml %}
  schema-registry:
    image: confluentinc/cp-schema-registry:5.5.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - zookeeper
      - broker
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: 'zookeeper:2181'
{% endhighlight %}

In our example, we use the following Schema Registry settings:
* `SCHEMA_REGISTRY_HOST_NAME`: The hostname advertised in ZooKeeper
* `SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL`: ZooKeeper URL for the Kafka cluster

## 3. Creating a Topic on Startup

We can create topics during the bootup sequence of the images, but we need to wait until Kafka is ready. To wait for Kafka to be ready, we can use a utility provided by Confluent: `cub kafka-ready`.

{% highlight yml %}
  create-topics:
    image: confluentinc/cp-kafka:5.5.0
    hostname: create-topics
    container_name: create-topics
    depends_on:
      - broker
    command: "
      bash -c 'cub kafka-ready -b broker:29092 1 120 && \
      kafka-topics --create --if-not-exists --zookeeper zookeeper:2181 --partitions 2 --replication-factor 1 --topic balance'"
    environment:
      KAFKA_BROKER_ID: ignored
      KAFKA_ZOOKEEPER_CONNECT: ignored
{% endhighlight %}

In our example, we wait for kafka to be ready providing the following arguments:
* `-b broker:29092`: Our bootstrap broker
* `1`: Minimum number of brokers to wait for
* `120`: Time in seconds to wait for the broker to be ready

## 4. Run Kafka with Compose

From our project directory, we can start up Kafka locally by running `docker-compose up`:

{% highlight shell %}
Creating network "kafka-interactive-queries_default" with the default driver
Creating zookeeper ... done
Creating broker    ... done
Creating create-topics   ... done
Creating schema-registry ... done
{% endhighlight %}

Compose starts the services that we defined and creates the topics.

We can stop Kafka, either by running `docker-compose down` from within our project directory in a second terminal or by hitting `CTRL+C` in the original terminal where we started Kafka.

## 5. Conclusion

We have seen an example of a _docker-compose.yml_ file to run Apache Kafka locally. Furthermore, we saw how to create topics on startup of Docker Compose. 

Running Kafka locally can be useful in simulating a productive environment for running integration tests. Especially with a microservice architecture, where microservices use messages to communicate with each other.

The full source code is available [on GitHub][github].

### References:

* [Get started with Docker Compose][docker-compose]
* [Confluent Platform Quick Start (Docker)][docker-quickstart]
* [Broker Configurations][broker-config]

[github]: https://github.com/casasprunes/tutorials/tree/master/kafka-interactive-queries
[docker-compose]: https://docs.docker.com/compose/gettingstarted/
[docker-quickstart]: https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html#ce-docker-quickstart
[broker-config]: https://docs.confluent.io/current/installation/configuration/broker-configs.html
