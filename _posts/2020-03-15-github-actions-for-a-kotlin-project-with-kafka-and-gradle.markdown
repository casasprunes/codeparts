---
layout: post
title:  "GitHub Actions for a Kotlin project with Kafka and Gradle"
date:   2020-03-15 19:40:00 +0100
tags: kotlin kafka gradle
---
We can easily set up a continuous integration (CI) workflow in GitHub Actions to build and test our Kotlin project with Gradle.
To do so, we need to create a file under `.github\workflows\workflow.yml`.

First, we need to define the name of the workflow and on what actions do we want to trigger it. In our case we want to trigger it every time we create a pull request for the master branch and every time we merge a pull request to the master branch. 

{% highlight yml %}
name: Build

on:
  pull_request:
    branches:
      - master
  push:
    branches:
      - master
{% endhighlight %}

Next, we have the jobs section with only one job `build` configured to run on Linux, using the GitHub-hosted `ubuntu-latest`.
The job consists of 5 steps which will in this order: check out the code from github, set up the JDK 1.8, start the docker containers for Zookeeper, Kafka and Schema Registry, grant permissions to execute gradlew and build and run the tests with gradle.

{% highlight yml %}
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Git branch
        uses: actions/checkout@v2
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Start Docker containers for Zookeeper, Kafka and Schema Registry
        run: docker-compose up -d
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Build and run Tests with Gradle
        run: ./gradlew build
{% endhighlight %}

GitHub provides predefined actions like `actions/checkout@v2` or `actions/setup-java@v1` that we can use out-of-the-box.

The full source code is available [on GitHub][github].

[github]: https://github.com/casasprunes/piggybox
