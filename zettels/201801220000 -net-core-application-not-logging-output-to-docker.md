
---
title: ".NET core application not logging output to Docker"
slug: "-net-core-application-not-logging-output-to-docker"
date: "2018-01-22"
summary: ""
---

#software-development #dotnet #devops

Since the introduction of .net core 2.0 everything on the .net platform seems to become better and better. Since the introduction of Visual Studio for mac it encouraged me to switch over to a mac OS X only dev environment.

![http://localhost:3000/uploads/versions/screen-shot-2018-01-22-at-11-21-27-am---x----588-659x---.png](/uploads/screen_shot_2018_01_22_at_11_21_27_am_x_588_659x_e41427cd87.png)

Needless to say my current development environment with Visual Studio, Docker, and Kitematic works amazing. And I only just started figuring things out.

The thing which has been bugging me though was that it seemed like application output (`stdout` and `stderr`) was not being logged to Docker while it was being logged to Visual Studio.

## **How to get an application to log to Docker**

Docker containers built from `microsoft/dotnet:2.0-runtime` or `microsoft/aspnetcore:2.0` have a `/remote\_debugger` volume which maps to `~/.vsdbg`. For now I can only imagine how this stuff works, but I suppose that when the remote debugger is attached the application only logs to the remote debugger which may or may not be favorable behavior.

In case you know more about the `/remote\_debugger` volume and the `~/.vsdbg` thingy or you can point me in the right direction, I'd love to hear more from you! Hit me up on [Twitter](https://twitter.com/CorstianBoerman)!
