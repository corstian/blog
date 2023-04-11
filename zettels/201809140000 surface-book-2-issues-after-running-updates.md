
---
title: "Surface Book 2 issues after running updates"
slug: "surface-book-2-issues-after-running-updates"
date: "2018-09-14"
summary: ""
---



This is definitely more of a note to myself, and my fellow Surface Book 2 users;

Lately mostly after installing Windows updates, the book is plagued by strange behavior.

A few things include:

* Difficulties to start up. When the power button has been pressed, it won't quickly start up after hibernation. Most of the times you have to press a few times in a row to get the Surface to wake up. Pressing the detach button mostly is the quickest way to wake up in this situation.
* After a sleep the NVIDIA GTX 1050/1060 isn't recognized after sleep. Detaching and then attaching may work to detect it again. A restart also works fine. So now and then a warning would pop up that there is a hardware problem with the base.
* Bad performance for detaching. It'd be easily possible to have to wait about 20 or 30 seconds for the base to detach. If at all. A restart mostly fixed these issues.

There is an old Reddit post I stumbled upon with people experiencing the same problem, and few fixes have been shared.

Go here for the Reddit post: https://www.reddit.com/r/Surface/comments/81qvp1/my\_surface\_book\_2\_having\_issues\_with\_waking\_up/

The top comment seemed to fix my problems. With the most recent updates I figured out that booting in UEFI mode (by pressing the power button and the volume up button) and exiting again solved my problems.
