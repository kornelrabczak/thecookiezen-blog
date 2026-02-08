---
layout: post
title: "JDD 2015 - Solutions worth adopting"
featuredImage: "/images/jdd.jpeg"
author: "Korneliusz Rabczak"
date: 2015-11-29 18:06:05 +0100
comments: true
description: JDD 2015 - Solutions worth adopting
keywords: kafka, rxjava, spock, ratpack, airomem, jdd, conference, java
categories: [java, conferences, solutions]
---



About a month ago I was at JDD conference which took place in Cracow. JDD is a two-day, Java focused, international based conference. There you have a chance to speak with many Java enthusiasts, exchange your experiences with attendees and speakers from all over the world. The conference was consisted of four parallel paths so everyone could find something interesting for himself. There were many technical lectures as well as soft topics, workshops and interactive trainings. I enjoyed the conference and I'm looking forward to the next edition.

<!-- more -->

Solutions from JDD lectures worth adopting or at least giving them a chance in a project that can handle the risk of the unknown :

*   **RxJava** - https://github.com/ReactiveX/RxJava - really nice library which is an abstraction for concurrency in Java world. It's a pleasure to create asynchronous and event-based applications using this implementation of observer pattern. RxJava introduces Observable model with the ease and simplicity. We can process streams of asynchronous events. You will be able to pay less attention to low-level threading, synchronization, thread-safety and focus more on business logic of your application.

*   **Spock framework** - http://spockframework.github.io/spock/docs/1.0/index.html  - great unit testing framework based on Groovy with a lot of syntactic sugar that will make writing test a lot easier and improve code clarity. Spock is easy to learn as well as to read. It will certainly increase your team productivity.

*   **Ratpack** - https://ratpack.io - simple toolkit, a set of open source libraries for creating high performance web applications written in any JVM language. Ratpack is build on top of Netty, asynchronous event-driven network application framework. Yet another fancy framework for building microservices.

*   **Airomem** - https://github.com/airomem/airomem - implementation of Prevalent System design pattern called also Memory Image. Main idea is to store data in memory and all writes journaled for system recovery instead of keeping everything in a database. Airomem is really great for prototyping and early phases of the project. We can easily follow Uncle Bob Clean Architecture rules where choice of data storage should be delayed because it's only a detail. I will write about this topic a separate blog post.

*   **Kafka** - http://kafka.apache.org/ - is one of possible solutions for event-driven microservices architecture. Kafka can act as message bus and participate in asynchronus communication between many microservices keeping them decoupled. Also solves scalability and availability problems.
