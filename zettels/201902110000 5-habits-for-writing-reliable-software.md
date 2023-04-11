---
title: "5 habits for writing reliable software"
slug: "5-habits-for-writing-reliable-software"
date: "2019-02-11"
summary: ""
---

#software-development

Practice shows that writing reliable software is extremely difficult. It's not as if writing code is so difficult. Even monkeys can write code. No, it's the edge cases that make it difficult, undocumented behavior, maybe even worse is unexpected behavior. Black boxes which do not show what's going wrong deep within the program, and when an error finally gets the change to bubble up the program crashes, with no trace left behind. I'll share with you 5 of my habits for writing reliable code;

## 1. Fail hard, fail fast

Basically it's the other way around. If anything goes wrong, try to fail fast, and then fail hard. Let your application be vocal about how it feels. Let it throw a tantrum when something goes wrong. A parent will probably be able to handle it.

Throw errors if you cannot handle it properly, rethrow errors whenever applicable, and please, [keep the stack trace intact](https://scottdorman.blog/2007/08/20/difference-between-throw-and-throw-ex-in-net/)!

In case you are not entirely sure what to do and the application (state) might be corrupted, don't feel bad to make the app crash. Crashing applications are usually better then running a crippled application which might cause continuous damage while running.

## 2. Avoid duplication

Code duplication also means bug duplication. Limit the amount of code you have and I promise you'll still be amazed by the amount of bugs in it. Besides that, it's just flat out annoying and embarrassing when you fixed a bug only to discover the bug still exists in the same code in a slightly different place.

One part of avoiding code duplication is making sure that you know where to find certain methods for certain tasks. There are too much occurrences where the same code has been written multiple times because the function was hidden in some obscure namespace or class.

Having duplicate code throughout the code base may be an indication that there's something wrong with your architecture. Go fix these issues!

## 3. Write automated tests

Small changes in code can have big impact throughout your application. As the application grows you will miss things, forget about pieces of code and lose details about the application. Automated tests will save your ass.

Tests are not the law though. If a test fails you can either change the tested method or the test function. In case of the latter, just be sure you know what you're doing!

If you're just starting out, and you're wondering how you test your code, a good begin is to think about all the small tasks your code has to do. Write tests which describe the desired output for your task, write your task and test it. Writing tests forces you to think about the purpose of a block of code. Doing so will generally result in better code reuse throughout the application.

## 4. Use logging

Writing stable code is amazing, but there will still be bugs. Part of stable code is minimizing the downtime and damage caused by bugs. In case there is a bug in a production environment (of course you're not testing in production), it's extremely beneficial to have a proper stack trace. Even better would be to have a log of events leading up to the bug so you quickly have an idea of what is going wrong.

Please be aware that there's such a thing as too much logging. Emitting hundreds of log messages a minute might be usable while debugging but it'll be a total disaster to spit through these in production. In addition this might result to real errors being missed due to the high volume of messages. And let's be honest, logging code scattered throughout your codebase is just ugly.

## 5. Prepare for the worst

Believe it or not, even if you apply all of this advice you will still encounter bugs in the worst possible of times. It's due to&nbsp;[Murphy's law](https://en.wikipedia.org/wiki/Murphy's_law):

> "Anything which can go wrong will go wrong"&nbsp;

[Calculate your risk](http://andrew.triumf.ca/cgi-bin/murphy.html), and prepare for the unexpected.

* Make sure your application will restart in case it shuts down.
* Implement retry logic in case some logic does not execute properly.
* Notify sysadmins proactively when unexpected behavior is detected (high load/response times/throughput etc)
* Limit potential damage / downtime caused by potential bugs
* Use continuous integration and continuous deployment (because hey, how many times has it already happened that you copied the wrong version to the wrong environment)

Writing code isn't hard. Writing stable software is.
