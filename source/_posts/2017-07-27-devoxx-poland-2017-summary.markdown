---
layout: post
title: "Devoxx Poland 2017 - Summary"
date: 2017-07-27 21:16:57 +0200
comments: true
description: Devoxx Poland 2017 - Summary
keywords: devoxx, conference, microservices, akka
categories: [conferences, devoxx]
---

-> {% img center-displayed /images/devoxx2017/devoxx.jpg 'Devoxx 2017' 'images' %} <-
                                                                                                                                                                                                                                  
After a 6 months break I’m back to blogging. I hope that my motivation will stay as high as it was when I was starting with it. My absence was caused by the operation of moving to Berlin, what was pretty occupying*. Now, when the moving is finally done, I can go back on the trail* :)

*After 2 months I still don’t have stable internet connection at home. Don’t ask me why.

Last month I was at a great JVM/Architecture/Cloud/Microservices related conference in Krakow - Devoxx Poland 2017. Not only did I meet my friends from previous companies over there, but I also attended in some rather interesting talks. As a re-warm-up blog post I’ve prepared a summary of some of those.
                                                                                                                                                                                                                                  
<!-- more -->

##Venkat Subramanian - Speed without discipline: a Recipe for Disaster

{% blockquote Venkat Subramaniam %}
Software development: a profession where people get paid to write poor quality code and get paid more later to cleanup the mess.
{% endblockquote %}

Venkat’s keynote was about current state of our IT industry. First part of the talk was focused on increasing interest of programmers to discover the advantages of functional programming and the declarative approach. Second part was devoted to testing. Venkat called software development without testing a “JDD” - Jesus Driven Development. Which means: write the code and pray that it’s working. We can only be confident to refactor, change, add new features when our code base is covered by well crafted tests. Otherwise we are doomed. 

A lot of companies in our industry struggle with automated testing of the UI. In his opinion, the project should have more low level unit tests then integration or UI tests. Unit tests are much faster, so running them is a lot cheaper than running GUI tests. Main idea of the Software Testing Pyramid is to define proper proportions between different types of tests in your system.

-> {% img center-displayed /images/devoxx2017/testingpyramid.jpg 'Software Testing Pyramid' 'images' %} <-

Venkat has pointed out that a lot of current solution for UI level testing is a pathway to hell. When designing and building your application you should carefully plan how your application will be tested, to not end with the ice-cream cone anti-pattern. A lot of projects these days focus more on having user interface tests than Unit test. The costs of having over engineered UI testing is: slow builds, complex deployment pipeline and no easy way to run them on local environment.

-> {% img center-displayed /images/devoxx2017/icecreamcone.jpg 'Ice-cream cone anti-pattern' 'images' %} <-

##Victor Rentea - The Art of Clean Code

Here we found a lot of references to Uncle Bob’s “Clean Code” book and video series. As a reminder, the most important best practices that we should follow to be the responsible software developers:
 
 - SRP
 - communicate, don’t code
 - Clean Code rules
 - KISS
 - measure, don’t guess
 - Boy Scout rule
 - SOLID
 - continuous refactoring - because when you stop, legacy will come and destroy your maintenance availability

Here are some of my thoughts after deliberating over beer with my old friends. It’s about the responsibility, or rather its lack in Software Development. We constantly must deal with people who just don’t want to be responsible developers at all. This problem is related to the experienced mid/senior developers, who know OOP principles and all of the things like SRP, SOLID etc. but they still rush with writing their code in a crappy way. I’ll write a separate blog post about it. For now, I’m referring to one of my favourite comic scenes and its main lesson:

**with great power comes great responsibility**

-> {% img center-displayed /images/devoxx2017/spiderman.jpg 'Great responsibility' 'images' %} <br/><sup>Marvel Spiderman: Amazing Fantasy #15</sup> <-

**Writing good code, that other developers can read and maintain in next years, is your responsibility, not a whim!**

##Paweł Barszcz - Kotlin - your 2017 Java replacement

-> {% img center-displayed /images/devoxx2017/kotlinlogo.jpg 'Kotlin logo' 'images' %} <-

Kotlin is a great language, it’s in Tiobe 50 ranking. It’s created and maintained by well known company - JetBreains. The language is something between Java and Scala. It’s for people tired of writing Java boilerplate code. Also for the ones that still have some fears to use Scala because there is a lot of magic around it. Kotlin seems to be a perfect option then. Another good-to-know thing is, that JetBrain lately became the part of the JCP committee. They will certainly bring some fresh and positive influence for future of the Java language.

Features that I liked:

 - Scala like syntax sugar, immutability of collections by default, data classes (value objects)
 - simple integration with Spring framework, and more Kotlin support for Spring 5
 - everything is final by default
 - null-safety the killer feature
 - easy usage with JSON
 - JetBrains Exposed abstraction for SQL
 - language that is easy to start, low risk for a project when starting without any experienced Kotlin developer (can't say the same about Scala), pragmatic approach
 - coroutines, type aliases, sealed classes, lazy sequences

##Johan Janssen - Using actors for The internet of (Lego) Trains

Johan was talking about amazing proof of concept project related with Internet of Things called The Internet of (Lego) Trains. The project was started to validate sense of using Scala and Akka stack for IoT purpose. Gathering data from a Raspberry Pi and camera, controlling trains and other elements of the infrastructure was built on top of the actor system with Scala, Akka, Akka HTTP and for web presentation AngularJS. For Akka HTTP load/performance testing an open source http://gatling.io/ was used, which has a nice Scala DSL. The presentation ended with a cool demo showing us how you can monitor and control Lego trains from web browser. It was a nice way to learn some new technology and make people within the company curious and creative.

##Jarek Ratajski - On @annotations - liberate yourselves from demons

Jarek’s talks are always controversial. Controversial in a good way. He is not afraid to criticize things that are widely used by most of the developers. This time he didn’t disappoint me as well. The talk was about negative influence of annotations to software maintainability and simplicity. All annotations originating from Java EE, Spring or other fancy framework are mostly redundant because instead of focusing on writing real Java code, we write a lot of code in a really weird way. Dependency injection using EJB or Spring container is harder to test when we have injection points in our whole code base. You don’t need any container for things like DI, you should use complex solution only for your high level layers to manage injections and transactions. Another problem are JPA annotations, big entity classes, too verbose - stringly typed etc. There are some usable annotations like @Override, @FunctionalInterface or @Deprecated but generally annotations have become the same evil and overused things as XML configuration files. 

[Read more](http://annotatiomania.com)

##Arun Gupta - Docker container orchestration platforms on Amazon

Arun, who not so long ago joined Amazon, was talking about running Docker containers on a Cloud. He showed us a few interesting examples of how to run different cluster management engines (Docker Swarm, Kubernetes, Mesos and Amazon ECS) on Amazon Cloud. From what I’ve learned, there is a really easy way to set up everything when you know how these things works. You can create new cluster and manage deploying application in fast and secure way. There is only one question: is it still so easy to run and maintain the large number of production clusters?

I need to find some more time to play with Kubernetes. At the last week’s Zalando Tech meetup about Docker,  there was an interesting topic about Kubernetes’ new feature called Ingress. Ingress API is more or less about integration custom load balancing server with Kubernetes clusters. Looks like I already have tens of ideas what to write next blog posts about!

##Daniel Rebrero - Applying stability patterns: a case study

Useful patterns from “Release IT” book presented with real life examples:

 - Use Timeouts - One of the most important thing is setting timeouts for everything. Don’t wait forever. Use timeouts for monitors, locks, etc.

 - Circuit Breaker for cascading failures, beware the service you don’t know. Stop 	doing it if it hurts.

 - Bulkheads - if one component fails, the rest of the services should work properly and not be affected, one application on each application server, allow partial failure.

 - Steady state - recycle resources, log rotation, cache size, DB/HTTP connections pools, pagination in API, data archive.

 - Thread pool - configure max/init size, bounded work queue.

 - Rejection policy - abort, back pressure, discard, discard oldest.

 - Fail Fast (don’t waste your time) - SLA check, circuit breaker status check, reserve resources, validate user input, monitor your dependencies.

 - Handshaking (agree before) - db connections pool, http connections pool.

 - Test Harness – WireMock, Docker, ToxiProxy, Arquillian Cube Q for chaos testing

 - Use Decoupling middleware, avoid waiting, read through cache, background fetch, write through cache, come back later, async IO non blocking server threads


##Vaughn Vernon - The Language of Actors

-> {% img center-displayed /images/devoxx2017/reactive.jpg 'Reactive Messaging Patterns with the Actor Model' 'images' %} <-

It was a very nice introduction to the high level concept of actors system. I've already bought Vernon's book “Reactive Messaging Patterns using Actors model” a week before. A lot of things from the first chapter were included in that presentation. Probably more valuable would be going for Vaughn Vernon’s workshops next time. The talk was focused on how actor model can help maintain the system resilience, handle failures and to stay responsive for our application users. He also briefly mentioned Domain Driven Design and how you can connect it with actor concept and messaging pattern as a communication between actors.

[Read more](https://www.lightbend.com/blog/designing-reactive-systems-with-the-actor-model-free-oreilly-book-by-hugh-mckee)


Attending this year’s Devoxx was not only a good excuse to see old friends. It was a nice opportunity to get inspired and stay updated with the latest IT trends in coding. Let’s see, what next year’s edition will bring us!
