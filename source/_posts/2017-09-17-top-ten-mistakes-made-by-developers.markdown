---
layout: post
title: "Top 10 mistakes made by developers on recruitment coding challenge"
date: 2017-09-17 20:04:57 +0200
comments: true
description: Top 10 mistakes made by developers on recruitment coding challenge
keywords: self improvement, recruitment, programming, best practices, clean code
categories: [self improvement, learning, recruitment, best practices]
---

-> {% img center-displayed /images/10mistakes/top10mistakes.jpg 'top10mistakes' 'images' %} <-

During the last few months I've been doing a lot of code reviews of simple code assignments that candidates get as a first recruitment task. This assignment requires basic programming skills like JAVA Collections API, design and some knowledge about best practices. The main purpose is to filter out developers who don't fit the company. Getting through such code doesn't seem entertaining. However, I came up with an idea to list 10 common mistakes. Make sure you avoid them next time when applying for a job position!

<!-- more -->

It's for everyone, regardless of experience in IT. Developers with 2, 5, 7, or even 10 years of experience in programming commit the same mistakes all over again. It's really disappointing to review a code of an 'experienced' developer who doesn't know basics like: DRY, SOLID, etc. You don't even need to read a book like Clean Code to have the basic feeling that SOMETHING in your code is not right.

There will be no good/bad examples, you can find everything easily explained on the internet. It's just a simple summary which can be used as a check list for the future.

Top 10 mistakes:

1. **Creating over-engineered architecture/design solution.**
 Remember about KISS - Keep It Simple Stupid! It's the most basic of the principles. Code readability is one of the most important things. If other developers can't read your code, you are doomed. You need to be able to write code that can be somehow maintained by other for next years.
2. **Not following well known and described best practices.**
 Not following naming conventions, providing misleading domain names.
3. **Using a language which you haven't been using for more than one year.**
 It's a trap, because on the face2face interview developers don't know how to write simple test case in their own application.
4. **Duplicating the same code everywhere over and over again.**
 Think about DRY - don't repeat yourself, refactor your code (extract method, object etc), remember also about keeping your tests code clean.
5. **Not following SOLID principles.**
  Lack of abstractions e.g DI, writing code that is hard to test. The Single Responsibility Principle violation, too much logic in single method, a lot of side effects. Lack of encapsulation, highly coupled classes and low cohesion are a sign of bad design.
6. **Not writing tests at all!**
  Lack of unit and acceptance tests. Too small tests coverage, empty methods in tests (setUp, tearDown), multiple assertions for single object - use equals method for object assertion.
7. **Trying to reinvent the wheel.**
  Implementing own LinkedList/Queue/etc. Providing own way for objects initialization - don't you know about objects constructor?!
8. **Using tools in a wrong way.**
  Not knowing tools that you're using, using some legacy tools like ANT or no tools for dependency management at all. Pushing all of the dependencies jar's into Git repository. Maven misconfiguration, test dependencies without test scope. Lack of Maven files structure convention, all files in the same package, no separation by features or domain.
9. **Not handling error cases correctly.**
 Pokemon exception handling. Losing original cause message and exception stack trace.
10. **Using Git in a wrong way.**
 Commits messages aren't meaningful. Commits are too large. Compiled application jar file in VCS repository. Commented code in VCS repository.

->  {% img center-displayed /images/10mistakes/tools.jpg 'Tools' 'images' %} <-

Remember, before you commit your next solution, make sure that you're following Clean Code, overall best practices, SOLID, DRY, KISS, DRY, KISS, AND DRY ONE MORE TIME! Make sure that you know how your tools work (build tool, dependency management, language, tests library, frameworks), know how to use GIT (c'mon... it's already 2017). Focus on simplicity, do not over-engineere your solution and last but not least: provide proper test coverage of your code.
