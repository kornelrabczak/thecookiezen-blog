---
layout: post
title: "Building Microservices awareness"
featuredImage: "/images/microservices/microservices.jpg"
author: "Korneliusz Rabczak"
date: 2018-01-07 17:24:00 +0100
comments: true
description: Building Microservices awareness
keywords: microservices, monolithic application, architecture, programming, best practices, clean code
categories: [microservices, architecture, best practices]
---



The idea of microservices is a really old one and the "microservices" buzzword got new life about 2 or 3 years
ago. Despite this, it's still something rather new for many people, who go crazy about using it at work. Because of these facts
you should think about building a proper microservices awareness in your company. As you don't want to build yet another distributed
system when you don't need it.

<!-- more -->

We can compare that, for example, to Scala "religious fanatics". When we ask some of these people why they choose Scala language, the first answer is usually "because it's cool". The same goes with microservices boom, as it's so cool to deal with all of the distributed systems problems... right? ...or not?

Before you start changing your Everything:
---------------------
Do you really need that? Or did you just read about microservices in some book? Maybe did you hear about it at one
of the conferences (where some part of the speakers share their knowledge with marketing extras involved)?

Your tools/solutions/patterns can be cool, but it shouldn't be the main reason you use them. The first and most important thing is, that your choice should allow you to build a proper and working solution to get your job done in the best/ good enough way. After all, you get money for that, not for playing with cool, fancy and popular solutions. Or playing darts. It's about being a professional engineer.

Are you aware of all of the trade-offs that microservices will bring to your head?
---------------------
It makes sense to break up your big domain based monolith system into smaller sub-domains running as
microservices. Or if there are too many people working on the same VCS repository. Just to lower merging and
personal conflicts.

It makes sense if you want to scale your components independently.

It can also help if you want to split builds and deployments. If they are slow and you want make
them independent from each other.

In other cases it probably doesn't make any sense. It can end up with:
---------------------
- all problems related with network failures that you can't even imagine at this point
- consumer driven contracts testing (read more about [PACT](https://docs.pact.io/)) to check if both consumer and producer are compatible
- maintaining and validating a schema for contracts
- a need of transactional deployments
- versioning your API's
- security issues (now when you share your API other teams may use it, are you ready for that?)
- stability patterns like bulkheads, back pressure, circuit breaker, retries
- idempotent resources
- service discovery
- the end of the world... or much worse ;)

Monolithic systems are good if they are well crafted using clean code, modularization and DDD approach. They can be easy to maintain and improve. It may look like splitting application into multiple services is the best solution to fix some of your problems but it's not necessarily. It's also hard to go back from this path, if you choose it at first.

If you want to start doing microservices, just remember that creating applications which are communicating over HTTP using JSON ... it's not doing microservices. It's probably a messy architecture and providing yourself nasty problems in the future. Learn more about [gRPC](https://grpc.io/), [protobuf](https://developers.google.com/protocol-buffers/), etc. before you start.

Do you really need microservices?
