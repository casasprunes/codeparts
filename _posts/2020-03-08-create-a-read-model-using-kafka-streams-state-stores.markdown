---
layout: post
title:  "Create a Read Model using Kafka Streams State Stores"
date:   2020-03-08 20:00:00 +0100
tags: kotlin kafka ratpack
---
Using Kafka Streams State Stores and Ratpack we can create a Read Model for our applications. Let's start by defining a stream that reads events from the `balance` topic and process them by using the `balanceReadModel` State Store.

{% highlight kotlin %}
// File: parts/code/piggybox/query/modules/KafkaModule.kt
val builder = StreamsBuilder()

val keyValueStoreBuilder: StoreBuilder<out KeyValueStore<String, out Any>> =
    Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore(config.stateStores.balanceReadModel),
        Serdes.String(),
        null as? Serde<*>
    )

builder.addStateStore(keyValueStoreBuilder)

builder
    .stream<String, SpecificRecord>(config.topics.balance)
    .process(ProcessorSupplier { processor }, config.stateStores.balanceReadModel)

val properties = Properties().apply {
    put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, config.bootstrapServersConfig)
    put(StreamsConfig.APPLICATION_ID_CONFIG, QueryServiceApplication::class.java.simpleName)
    put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String()::class.java)
    put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, SpecificAvroSerde::class.java)
    put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000)
    put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
    put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, config.schemaRegistryUrlConfig)
    put("value.subject.name.strategy", TopicRecordNameStrategy::class.java)
}

return KafkaStreams(builder.build(), properties)
{% endhighlight %}

The processor waits for Domain Events to happen and updates the Read Model accordingly. 
As we can see in the following example, each time we process a `FundsAdded` event for a customer, this customer's Balance is increased and stored in the Read Model.

{% highlight kotlin %}
// File: parts/code/piggybox/query/streams/suppliers/RecordProcessor.kt
class RecordProcessor @Inject constructor(
    private val config: KafkaConfig
) : Processor<String, SpecificRecord> {

    private lateinit var state: KeyValueStore<String, BalanceState>

    override fun init(context: ProcessorContext) {
        state = context.getStateStore(config.stateStores.balanceReadModel) as KeyValueStore<String, BalanceState>
    }

    override fun process(key: String, record: SpecificRecord) {
        when (record) {
            is FundsAdded -> {
                val balanceState = state.get(record.customerId)
                val newBalance = (balanceState?.amount ?: BigDecimal.ZERO) + record.amount

                state.put(record.customerId, BalanceState(record.customerId, newBalance, record.currency))
            }
        }
    }

    override fun close() {}
}
{% endhighlight %}

Our Kafka Streams run as a service inside a Ratpack application. We can also use Ratpack to add a Remote Procedure Call (RPC) layer to our application. In this way, we can query the Kafka Streams State Store as a Read Model. 

{% highlight kotlin %}
// File: parts/code/piggybox/query/QueryServiceApplication.kt
object QueryServiceApplication {

    @JvmStatic
    fun main(args: Array<String>) {
        RatpackServer.start { server ->
            server
                .serverConfig { config ->
                    config
                        .baseDir(BaseDir.find())
                        .yaml("application.yaml")
                        .require("/kafka", KafkaConfig::class.java)
                        .jacksonModules(KotlinModule())
                }
                .registry(Guice.registry { bindings ->
                    bindings
                        .module(KafkaModule::class.java)
                        .module(QueryModule::class.java)
                        .module(APIModule::class.java)
                        .bindInstance(ObjectMapper::class.java, ObjectMapper().registerModule(KotlinModule()))
                })
                .handlers { chain -> chain.prefix("api", APIEndpoints::class.java) }
        }
    }
}
{% endhighlight %}

We have only one endpoint `api/customers.getBalance` which given a JSON payload with the `customerId`, searches the Read Model and returns the customer's Balance.

{% highlight kotlin %}
// File: parts/code/piggybox/query/api/APIEndpoints.kt
class APIEndpoints : Action<Chain> {

    override fun execute(chain: Chain) {
        chain
            .get("customers.getBalance", CustomersGetBalanceHandler::class.java)
    }
}
{% endhighlight %}

{% highlight kotlin %}
// File: parts/code/piggybox/query/api/handlers/CustomersGetBalanceHandler.kt
class CustomersGetBalanceHandler @Inject constructor(
    private val config: KafkaConfig,
    private val streams: KafkaStreams
) : Handler {

    override fun handle(ctx: Context) {
        ctx.parse(CustomersGetBalancePayload::class.java).then {
            val store = streams.store(
                config.stateStores.balanceReadModel,
                QueryableStoreTypes.keyValueStore<String, BalanceState>()
            )

            ctx.response.status(Status.OK)
            val record = store.get(it.customerId)
            ctx.render(Jackson.json(BalancePayload(record.amount, record.currency)))
        }
    }

    private data class CustomersGetBalancePayload(val customerId: String)
    private data class BalancePayload(val amount: BigDecimal, val currency: String)
}
{% endhighlight %}

A Read Model built in this way could safely be discarded and rebuilt from scratch whenever it has to change, due to bugs or new requirements. 
As a consequence of our `balance` topic being treated as an Event Store and having infinite retention, we could create a new read model at any time and it would retroactively include everything from the past.

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
