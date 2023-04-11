---
title: "Luisterbieb Windows Phone App"
slug: "luisterbieb-windows-phone-app"
date: "2014-08-24"
summary: ""
references: 
---

#software-development #dotnet

Recently I’ve been working on a ‘luisterbieb’ application for Windows Phone. ‘luisterbieb’ means something like audio library in Dutch. The main point behind this application is that users should be able to listen to many well known books. (There are around 300 available right now.)

This application is already available for the Android and iOS platforms. The main reason for me to develop a ‘clone’ for the Windows Phone platform is due to me always having my Windows Phone with me while traveling, and not an iOS or Android device.

![http://localhost:3000/uploads/versions/898a457a-03cf-45ec-8378-bc4f6868e79e---x----1024-629x---.png](/uploads/898a457a_03cf_45ec_8378_bc4f6868e79e_x_1024_629x_04dc050ba6.png)

I’ve almost completed this application. I can navigate through books, I can select which books I want to listen and so on, but, there’s one thing which prevents me from finishing this application.

When I started this application I thought it’d be as simple as shit because I had access to their API and I saw the direct links to their MP3 files were available. I tried to listen to an excerpt which worked. HOWEVER, it turned out these developers have applied some kind of encryption on their actual MP3 files (not the excerpts) which means I can’t play these files.

![http://localhost:3000/uploads/versions/2a993d60-1962-482b-93de-afa9189a7a92---x----372-768x---.jpg](/uploads/2a993d60_1962_482b_93de_afa9189a7a92_x_372_768x_4d15e079a1.jpg)

At the time of publishing I’ve just mailed the developers of this application with the request to help me play this file. Of course they have their reasons to encrypt these sound files. One of them is to prevent it from leaking to the web. Whatsoever, I dare to bet Windows Phone is at least more safe than the Android platform, and maybe even the iOS platform.

There are two possible outcomes on my request:

They deny and the development of the app will stop. They accept and I can finish this application, and publish it to the store. I truly hope the last is going to happen. Maybe in the first place so I can listen to these books when I’m traveling, and at second so I can publish a nice application to the Windows Phone store most of the Dutch Windows Phone users can use.

Let’s hope for the best!

*EDIT (2015):*

Despite the initial enthousiasm for making it possible to release this application the people from the body which nationwide governs the libraries have decided not to help with this application by providing the encryption keys. There were three problems (in chronological order):

* At first I didn't have a legal entity which is required for a contract between me and the governing body.
* At the time I could use a legal entity the body was reorganizing it's organisation.
* After the reorganization there were a lot of things which had higher priorities.
* Now in the summer of 2015 (Roughly a year later) they have expressed not to help with this project because they are actively developing a new (long planned) version of the application which means I'd have to update the application. Sadly they are not keen on providing documentation on their new API.

*Maybe I'll take another try when the application is released and there is no Windows (Phone) version yet.*
