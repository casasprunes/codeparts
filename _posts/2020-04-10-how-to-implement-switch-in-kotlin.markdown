---
layout: post
title:  "How to implement switch in Kotlin"
date:   2020-04-10 17:00:00 +0100
tags: kotlin
excerpt: "The equivalent of the switch statement in Kotlin is the when expression. Let's see an example of how we can use it."
---
The equivalent of the `switch` statement in Kotlin is the `when` expression. 

When compared with Java or Groovy, these are for me some ways in which the Kotlin `when` is better:
* a more compact syntax
* can be used without an argument
* smart casts
* we can use arbitrary expressions (not only constants) as branch conditions
* can be used as an expression (e.g. returning its value or assigning it to a variable)

Let's see an example of how we can use it for stream processing:

{% highlight kotlin %}
// File: preferences-service/src/main/kotlin/parts/code/piggybox/preferences/streams/suppliers/RecordTransformer.kt
override fun transform(key: String, record: SpecificRecord): KeyValue<String, SpecificRecord>? {
    val result = when (record) {
        is CreatePreferencesCommand -> transform(record)
        is ChangeCountryCommand -> transform(record)
        is AddFundsCommand -> transform(record)
        is BuyGameCommand -> transform(record)
        else -> unknown()
    }

    logger.info(
        "Transformed ${record.schema.name} to ${result.value.schema.name}" +
                "\n\trecord to transform: $record" +
                "\n\ttransformed to: $result"
    )

    return result
}
{% endhighlight %}

Stream processing is the continuous processing of data directly as it is produced or received, for this type of use case the `when` expression is especially useful because we are processing records to apply some transformation to them depending on their type and then returning the result.

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
