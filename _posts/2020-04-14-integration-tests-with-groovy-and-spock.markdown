---
layout: post
title:  "Integration Tests with Groovy and Spock"
date:   2020-04-14 22:00:00 +0100
tags: Kotlin Groovy
excerpt: "For creating Integration Tests with a BDD approach, using Spock with Groovy and JGiven, I think is an excellent choice."
card: "summary_large_image"
image: "/assets/integration-tests-with-groovy-and-spock.png"
author: "casasprunes"
---
I have a lot of fun coding in Kotlin, so usually, especially in my spare time, I try to use it for everything.

When I started [piggybox][github], I decided to use Kotlin, with JUnit 5 and JGiven for the Integration Tests. JGiven gave me the [BDD approach][bdd] that I wanted, making the tests behaviour-focused and human-readable and Kotlin gave me the fun of being able to use Kotlin's cool features.

But some minor details were bothering me, like not being able to use the *$* symbol in the method names. JGiven uses the *$* symbol to generate reports, and Kotlin didn't allow me to use it, but I found a workaround, I could use the *@As* annotation to add any description I wanted for the reports, and in here it was allowed to use the *$* symbol.

Then I came across another annoying detail, JGiven has a method called *when()* that I needed to use to write the tests in a BDD style, but *when* is a reserved keyword in Kotlin so I couldn't use it. Fortunately, I found another workaround, we can add quotes around it, and it works, but then the code looks messy as we can see in the following example:

{% highlight kotlin %}
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

I could live with those two minor nuisances, but next, I wanted to add a data-driven test. At this point, I was looking into different test frameworks for Kotlin to see what were the best options, but I couldn't find anything that was cleaner or more expressive than Spock.

I have used Spock before; the problem is that Spock is for Java and Groovy, and I wanted to do this project in Kotlin, but wouldn't that be falling into the golden hammer anti-pattern, trying to use the same tool for everything? Especially when there are problems that could be solved better with other tools. 

For creating Integration Tests with a BDD approach, using Spock with JGiven, I think is an excellent choice. And I'm ok mixing Groovy with Kotlin, Groovy for the Integration Tests with Spock and JGiven and Kotlin for the rest. Being both of them JVM languages, they work well together.

Here we can see how the tests look right now using Spock data tables:

{% highlight groovy %}
// File: integration-tests/src/test/groovy/parts/code/piggybox/integration/tests/features/AddFundsShould.groovy
@Unroll
def "not add #fundsToAdd funds if new balance is greater than 2000"() {
    expect:
    given().applicationsUnderTest(applicationsUnderTest)
           .customer_preferences_with_currency_$_and_country_$("EUR", "ES")
           .and().$_$_worth_of_funds(originalFunds, "EUR")
    when().adding_$_$_worth_of_funds(fundsToAdd, "EUR")
    then().$_$_worth_of_funds_are_denied(fundsToAdd, "EUR", Topics.balanceAuthorization)

    where:
    originalFunds | fundsToAdd
    0.00          | 2000.01
    1000.00       | 1000.01
    2000.00       | 0.01
}
{% endhighlight %}

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
[bdd]: https://code.parts/2020/03/28/testing-with-jgiven-and-kotlin/
[github]: https://github.com/casasprunes/piggybox
