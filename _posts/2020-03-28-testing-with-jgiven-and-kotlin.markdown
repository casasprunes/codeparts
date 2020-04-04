---
layout: post
title:  "Testing with JGiven and Kotlin"
date:   2020-03-28 13:00:00 +0100
tags: kotlin jgiven
excerpt: "JGiven helps us to create tests with a BDD approach. It gives us the necessary tools to build our own fluent, domain-specific API."
---
JGiven helps us to create tests with a BDD approach. It gives us the necessary tools to build our own fluent, domain-specific API and defines a clear way of writing tests divided into three sections:
* _given_ some state   
* _when_ some action   
* _then_ some outcome   

The tests are written in a human-readable language that focuses on behavior, as a result, the tests are more accessible to domain experts, testers and developers.

In the following example we see how we can use it:

{% highlight kotlin %}
// File: integration-tests/src/test/kotlin/parts/code/piggybox/integration/tests/features/AddFundsFeature.kt
@Test
fun `should add funds to a customer`() {
    given()
        .applicationsUnderTest(applicationsUnderTest)
        .customer_preferences("EUR", "ES")
    `when`()
        .adding_funds(1.0, "EUR")
    then()
        .the_funds_are_added(1.0, "EUR")
        .and().the_customer_balance_is(1.0, "EUR")
}
{% endhighlight %}

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
