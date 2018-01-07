---
layout: post
title: "Building Microservices awareness"
date: 2018-01-07 17:24:00 +0100
comments: true
description: Top 10 mistakes made by developers on recruitment coding challenge
keywords: microservices, monolithic application, architecture, programming, best practices, clean code
categories: [microservices, architecture, best practices]
---

-> {% img center-displayed /images/microservices/microservices.jpg 'microservices' 'images' %} <-

The idea of microservices is really old, and the "microservices" buzzword got new life about 2 or 3 years ago. But it's still something new for many people which are crazy about using it at work. Because of that you should build proper microservices awareness in your company to not build yet another distributed system when you don't need it.

<!-- more -->

We can compare that for example to Scala "religious fanatics". When we ask some of those people why they choose Scala language, the first answer is "because it's cool". It's the same with microservices boom, it's so cool to deal with all of the distributed systems problems... right? ...NOT!

Do you really need that? Or did just read about microservices in some book or you heard about it on one of the conferences (where most people are sharing knowledge but there is also a lot of marketing)? Your tools/solutions/patterns should not be a cool things at first. First and most important thing is your choice should allow you to build proper and working solution to get your job done in the best/right way. Because you're getting money for that, not for playing with cool, fancy and popular solutions on your production using your employer money. It's about being professional engineer.

Are you aware about all of the trade-offs that microservices will bring on your head?

It makes sens to break up your big domain based monolith system into smaller sub-domains running as microservices. Or if there is to many people working on the same VCS repository. To lower merging and personal conflicts.

It makes sens if you want scale your components independently.

It makes also help if you want to split builds and deployments. Because they are slow and you can make them independent.

In other cases it probably doesn't make any sens. And if you do that you will need to deal with:
- all problems related with network failures that you can't even imagine
- consumer driven contracts testing (read more about [PACT](https://docs.pact.io/)) to check if both consumer and producer are compatible
- maintain and validate schema for contracts
- maybe you need transactional deployments
- versioning your API's
- security (now when you share your API other teams may use it, are you ready for that?)
- stability patterns like bulkheads, back pressure, circuit breaker, retries
- idempotent resources
- service discovery
-... and many more ;)

Monolithic systems are good if they are well crafted using clean code, modularization and DDD approach. They can be easy to maintain and improve. It may look like splitting application into multiple services is the best solution to fix some of your problems but it's not and it will be hard to go back.

If you want to start doing microservices just remember that creating applications which are communicating over HTTP using JSON ... it's not doing microservices, it's probably a mess architecture and asking yourself for problems in future. Learn more about [gRPC](https://grpc.io/), [protobuf](https://developers.google.com/protocol-buffers/), etc. before you start.

Do you really need a microservices? Or you just want to have fun.
