---
layout: post
title:  "Money and Currency Library for Kotlin"
date:   2020-04-24 22:00:00 +0100
tags: Kotlin
excerpt: "In this tutorial, we'll learn about a simple Money and Currency library that we can use with Kotlin so that we don't have to create our own."
card: "summary_large_image"
image: "/assets/money-and-currency-library-for-kotlin.png"
author: "casasprunes"
---
## 1. Introduction

When working on fintech applications, we sometimes find ourselves needing to create our Money class.
In this tutorial, we'll learn about a simple Money and Currency library that we can use with Kotlin so that we don't have to create our own.

## 2. Installation

### 2.1 Maven Dependency

We need to add the dependency into the *pom.xml* file:

{% highlight xml %}
<dependency>
    <groupId>parts.code</groupId>
    <artifactId>money</artifactId>
    <version>1.0.0</version>
</dependency>
{% endhighlight %}

Since it's not on Maven Central, we have to enable the JCenter repository:

{% highlight xml %}
<repository>
    <id>jcenter</id>
    <name>jcenter</name>
    <url>https://jcenter.bintray.com</url>
</repository>
{% endhighlight %}

### 2.2 Gradle Dependency

We need to add the dependency into the *build.gradle* file:

{% highlight gradle %}
dependencies {
    compile 'parts.code:money:1.0.0'
}
{% endhighlight %}

Since it's not on Maven Central, we have to enable the JCenter repository:

{% highlight gradle %}
repositories {
    jcenter()
}
{% endhighlight %}

## 3. Getting Started

The goals of this library are: 
* To put *amount* and *currency* together into a data class
* To be able to perform operations with Money instances using symbolic operators
* To have the safety of not performing operations between different currencies
* To have an enum with the active alphabetic codes of official ISO 4217 currency names

It's a simple library that tries to do one thing and do it well, instead of trying to solve many problems. 

### 3.1 Money class

The Money class is an immutable representation of a money amount with currency, such as *USD 8.13*. We can instantiate it with amounts in different types *Int*, *Double*, *BigDecimal* and *String*.

{% highlight kotlin %}
val moneyFromInt = Money(11, USD)
val moneyFromDouble = Money(0.11, USD)
val moneyFromBigDecimal = Money(BigDecimal.valueOf(0.11), USD)
val moneyFromString = Money("0.11", USD)
{% endhighlight %}

We can also instantiate it by using the *zero* method:

{% highlight kotlin %}
val money = Money.zero(USD)
{% endhighlight %}

### 3.2 Arithmetic Operations

The Money class uses operator overloading to provide implementations for the arithmetic operators sum, subtract, multiply and divide, allowing us to perform operations in a simple way, without having to call any methods:

{% highlight kotlin %}
val moneySum = Money(0.11, USD) + Money(2.35, USD)
val moneySubtract = Money(0.11, USD) - Money(2.35, USD)
val moneyMultiply = Money(0.11, USD) * 3
val moneyDivide = Money(0.11, USD) / 3
{% endhighlight %}

Having the safety of not performing operations between different currencies:

{% highlight kotlin %}
assertThrows<IllegalArgumentException> {
    val moneyException = Money(0.11, USD) + Money(2.35, EUR)
}
{% endhighlight %}

### 3.3 Comparison

We can compare amounts of money by using the comparison operators:

{% highlight kotlin %}
val isGreater = Money(2.35, USD) > Money(0.11, USD)
val isGreaterOrEquals = Money(0.11, USD) >= Money(0.11, USD)
val isLower = Money(0.11, USD) < Money(2.35, USD)
val isLowerOrEquals = Money(0.11, USD) <= Money(0.11, USD)
val isEquals = Money(0.11, USD) == Money(0.11, USD)
val isNotEquals = Money(0.11, USD) != Money(2.35, USD)
{% endhighlight %}

### 3.4 Rounding

We can use the *round* method to round an amount of money by a scale and rounding mode:

{% highlight kotlin %}
val moneyRound = Money(0.112358, USD).round(3, RoundingMode.HALF_EVEN)
{% endhighlight %}

## 4. Summary

In this article, we've covered the basics of the Money and Currency Library for Kotlin. 

It's a simple library that can fulfil our needs if we just need to do some basic money calculations in our application.

The full source code is available [on GitHub][tutorial].

[tutorial]: https://github.com/casasprunes/tutorials/tree/master/kotlin
