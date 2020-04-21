---
layout: post
title:  "Building an HTTP application using Kotlin"
date:   2020-03-22 22:45:00 +0100
tags: kotlin ratpack
excerpt: "Using Kotlin along with Ratpack we can easily create a self-hosted server application that accepts HTTP requests."
---

Using Kotlin along with Ratpack we can easily create a self-hosted server application that accepts HTTP requests. 
 
Ratpack is a set of Java libraries that facilitate fast, efficient, evolvable and well-tested HTTP applications. It is built on the highly performant and efficient Netty event-driven networking engine.

To start using Ratpack we need to create a *build.gradle* file with the following contents:

{% highlight gradle %}
// File: command-service/build.gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "io.ratpack:ratpack-gradle:1.7.6"
    }
}

apply plugin: 'io.ratpack.ratpack-java'

mainClassName = 'parts.code.piggybox.command.CommandServiceApplication'

dependencies {
    implementation "io.ratpack:ratpack-guice:1.7.6"
}
{% endhighlight %}

Ratpack provides integration with Google Guice, this integration gives us a way to decouple the components of our application. We can factor out the functionality of each API endpoint into its own *Handler*. This makes our code more maintainable and more testable. It's the standard "Dependency Injection" pattern.

By extending Guice's *AbstractModule* we are overriding the default *configure* method to create bindings for all our handlers.

{% highlight kotlin %}
// File: command-service/src/main/kotlin/parts/code/piggybox/command/modules/WebAPIModule.kt
class WebAPIModule : AbstractModule() {
    override fun configure() {
        bind(WebAPIEndpoints::class.java)
        bind(AddFundsHandler::class.java)
        bind(BuyGameHandler::class.java)
        bind(ChangeCountryHandler::class.java)
        bind(CreatePreferencesHandler::class.java)
    }
}
{% endhighlight %}

As we can see in the previous file we are also creating a binding for the *WebAPIEndpoints* class, this class is a Ratpack *Action* that is used to map the endpoints of our Web API with the specific handlers.

{% highlight kotlin %}
// File: command-service/src/main/kotlin/parts/code/piggybox/command/api/WebAPIEndpoints.kt
class WebAPIEndpoints : Action<Chain> {
    override fun execute(chain: Chain) {
        chain
            .post("preferences.create", CreatePreferencesHandler::class.java)
            .post("preferences.changeCountry", ChangeCountryHandler::class.java)
            .post("balance.addFunds", AddFundsHandler::class.java)
            .post("balance.buyGame", BuyGameHandler::class.java)
    }
}
{% endhighlight %}

Finally, we can create the *CommandServiceApplication* object, with the following contents:

{% highlight kotlin %}
// File: command-service/src/main/kotlin/parts/code/piggybox/command/CommandServiceApplication.kt
object CommandServiceApplication {
    @JvmStatic
    fun main(args: Array<String>) {
        RatpackServer.start { server ->
            server
                .serverConfig {
                    it
                        .baseDir(BaseDir.find())
                        .jacksonModules(KotlinModule())
                }
                .registry(Guice.registry {
                    it
                        .module(WebAPIModule::class.java)
                        .bindInstance(ObjectMapper::class.java, ObjectMapper().registerModule(KotlinModule()))
                })
                .handlers {
                    it
                        .prefix("api", WebAPIEndpoints::class.java)
                }
        }
    }
}
{% endhighlight %}

This will be the entry point of our Web API, we can start it either by executing the run task with Gradle *./gradlew run* on the terminal, or by executing it from our IDE. 

After executing the application, the server will be available via *http://localhost:5050/* and will listen to HTTP calls to the following endpoints:   

| ENDPOINT |
| ------------------------- |
| http://localhost:5050/api/preferences.create |
| http://localhost:5050/api/preferences.changeCountry |
| http://localhost:5050/api/balance.addFunds |
| http://localhost:5050/api/balance.buyGame |           

In the next post, we will take a look at the implementation of the specific handlers.

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
