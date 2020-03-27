---
layout: post
title: "Never Ending Story of Delivery"
description: Never Ending Story of Delivery
keywords: architecture, programming, best practices, agile
categories: [agile, architecture, best practices]
date: 2019-01-06 11:53:22 +0100
comments: true
---

-> {% img center-displayed /images/smallstories/split1.jpg 'smallstories' 'images' %} <-

The problem
====================

There’s a never-ending story. It’s still not ready for shipping after working for over 3 sprints on it. Your project manager asks you all the time whether the job is going to be finished within this millennium. Your product owner asks you at every single meeting: why did the new feature not go live yet. Sounds familiar?

This unhealthy situation decreases team morale. We will rush with design decisions, work longer hours, we may have a feeling of not getting a closure ever, and failing our team. In the end all these elements have negative impact on us. And we end up with random, uncaught bugs.

{% blockquote Yoda %}
Fear is the path to the dark side. Fear leads to anger. Anger leads to hate. Hate leads to suffering.
{% endblockquote %}

<!-- more -->

It’s nothing new, I’m not writing about some rocket science, nor was Yoda. Most of us already know how to work in the most efficient way, yet a lot of teams fail because of not following the basic rules. I listed them here for you.

The root cause
====================

The most common reason why we fail with fast value delivery is the size of the feature story. If our feature story has an unbelievable size, the whole team is to blame. It means that everyone (from managers to devs) slept during the grooming/planning and no one woke up to the sound of the monstrous in size story approaching.

Having a proper team discussion about every story is crucial if we want to build a successful team. Finding hidden complexity, splitting big stories, clarifying acceptance criteria are essential to be agile. Still so many teams fail to do it right.

The main issue within the main problem of not delivering The Thing On Time can be still summed up to three words: too big stories. But watch out! You can end up with a failure, if you split the story in a wrong way. The order of your tasks also matters.

Case study
====================

Let’s have some example: simple HTML form implementation with few input selects/fields. The form should reload to prefill next input with predefined values when previous element changes. Submitting the form leads to the downloading a file.

List of tasks that we need to implement:

* a contract between backend and frontend applications to make sure that they know how to communicate with each other (etc. JSON schema)
* HTML form presentation
* JavaScript logic for form to reload on any input change
* download button
* backend endpoint to provide all necessary data to render a form
* backend endpoint to stream a file


Let’s start with The Wrong Approach
---------------------

We start from defining a contract. Then we could have 2 backend tickets for both endpoints and one frontend ticket to implement an HTML form and reload it on change. Sounds super simple, right?

It’s not. If people start working on things in a wrong order, it will become chaos at the end. Trying to integrate working frontend and backend will be the first big surprise. Why? People didn’t talk to each other. The assumptions were not right. Not even identical.

Lesson? Don’t make assumptions. Clarify everything. The project doesn’t need to be surrounded with question marks.

After fixing all of the integration issues, we would be super happy. Would… because now we need to test it. We’ll able to start testing our solution only when integration is done.

Working on a big story / splitting story in a wrong may end with:

* slow delivery
* late integration
* more bugs in the last stage
* no user feedback
* angry managers
* coffee addiction



-> {% img center-displayed /images/smallstories/split2.jpg 'smallstories' 'images' %} <-

Order matters
---------------------

To make your team life easier, you should try to focus on the tickets order. It’s very important to integrate both parts (frontend and backend) as soon as you can. You will find all of the countless issues much earlier. If you can make integration between frontend and backend on the first day, you have already something which is working. You can even show it to your PO/PM on the next day.

First working feature could be HTML form with a single download button and a dummy endpoint, which provides empty file on a backend side.

If you continue on working like that after one week, you are already able to show the minimum working solution to your customer on a demo.

Size matters
---------------------

The second improvement is to have smaller stories. Yes, it’s that simple. Try splitting your feature story in a way that each smaller story can be released separately. Try to find your minimum viable product and the next stories will be about improving it and adding more functionality and user value. It’s so much easier to estimate smaller stories. Your team will feel a sense of the achievement. Build your championship with small but systematic winnings.

Most important takeaways
====================

You should consider splitting your stories into small and deliverable chunks of work for a fast user feedback, and early integration.

The ability to split stories into small, deliverable values is important. Smashing important.

An early integration lets you catch the bugs sooner.

Fast delivery of your MVP leads to the early, valuable user feedback (and they always have something to say...)

And remember: As a developer your responsibility is to deliver the values to the user. Your company pays you some gold for that. Let’s keep them affording to do so.
