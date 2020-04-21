---
layout: post
title:  "Hide steps from JGiven report"
date:   2020-04-04 17:00:00 +0100
tags: kotlin jgiven
excerpt: "Steps can be completely hidden from the JGiven report by using the @Hidden annotation. Useful for technical methods, which should not appear in the report."
---
For me, the main advantage of using JGiven is the fact of being able to write tests in Kotlin and in a human-readable language that focuses on behavior. But JGiven has other interesting points such as reporting. Every time we run the tests, JGiven generates reports that are readable by domain experts.

Such reports can serve as living documentation but they mustn't include technical implementation details. Let's see for example one of our tests:

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

As we can see, it makes use of a method called *applicationsUnderTest* which is used to pass to the different stages a reference to the applications under test. This call is more of a technical necessity for the tests to function than behavior that must be tested. Therefore we are not interested in it appearing in the reports.

For these cases JGiven offers us the option to hide steps using the *@Hidden* annotation as follows:

{% highlight kotlin %}
// File: integration-tests/src/test/kotlin/parts/code/piggybox/integration/tests/features/stage/Given.kt
@ProvidedScenarioState
lateinit var applicationsUnderTest: ApplicationsUnderTest

@Hidden
open fun applicationsUnderTest(applicationsUnderTest: ApplicationsUnderTest): Given {
    this.applicationsUnderTest = applicationsUnderTest

    return self()
}
{% endhighlight %}

When doing so, the *applicationsUnderTest* method will be completely hidden from the reports, as we can see below:

{% highlight txt %}
 Should add funds to a customer

   Given customer preferences with currency EUR and country ES
    When adding 1.0 EUR worth of funds
    Then 1.0 EUR worth of funds are added
     And the customer balance is 1.0 EUR
{% endhighlight %}

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
