
---
title: "How code quality positively impacts the accuracy of estimates"
slug: "estimate-accuracy-and-code-quality"
date: "2022-03-24"
summary: ""
---

#software-development #web-development

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">A student asked me yesterday why orgs demand estimates of impossible-to-estimate activities. Best answer I could come up with was that an estimate allowed them to deflect blame. Someone yells at them for going too slow, they say, &quot;Yell at them. They estimated it.&quot;</p>&mdash; Kent Beck 🌻 (@KentBeck) <a href="https://twitter.com/KentBeck/status/1506654258212458501?ref_src=twsrc%5Etfw">March 23, 2022</a></blockquote>

Personally I'm not really fond of estimates anyway, therefore consider this post to be a rant against estimates as most of you know it. Especially within the software development industry estimates are known to be wild guesses which are highly inaccurate. Though widely known, this factor is rarely explicitly accounted for. The apparent inability to face the uncertainty and the resulting illusory sense of certainty perhaps make up the most widely believed lie in the industry.

The usual estimates serves as the containment of any estimation bias and the potential for the personification of these whenever deemed appropriate. This estimation bias may contain multiple components among which are the expected capacity, velocity, and resource requirements. All of these are characterized by the difficulty in pre-emptively recognizing the potential but unknown uncertainties. An additional complexity herein is are the ripples and cascading effects one (unknown) uncertainty may have on dependent components of the estimate.

To reduce the risk associated with unforseen risks one might require to provide more transparency in the areas where the unknown risks are most likely to originate from. Within the software development industry this area mostly revolves around interactions with the actual code.

Though how comes that the main thing we are supposed to do also bears the highest amount of risk? Within a mature field, shouldn't we be able to provide accurate estimates for the work we're about to do. I empathesize this is a difficult thing to do for experimental work, but at least we should be able to do so for the most common types of software related work. 

One strategy to reduce the inaccuracy of estimates is to play a game of [planning poker](https://en.wikipedia.org/wiki/Planning_poker). The principle is to scale the estimation process and letting multiple people provide estimates at the same time. A huge difference in estimates proves the use of different assumptions as basis for the estimation. These differences then can be used as input for a conversation wherein the goal is to make these implicit assumptions explicit, and either to take them into consideration or to consider them unfounded. Practically this will usually lead to a conversation discussing the reasons for under or over-estimating.

Although the practice of playing planning poker allows the team to converge onto an estimate, the same underlying causes for inaccuracies still exist. Whenever an estimate is wildly inaccurate this isn't so much the shortcoming of a developer or a team, but rather a symptom of the environment they are working in. The sole fact that complexity in a process is able to hide in unforseen ways tells two things about the codebase:

1. There is something wrong with the abstractions and genericity of the codebase.
2. The processes of the work to be done are not properly understood.

Even though I am not a fan of formally documented development or implementation processes, having a high-level overview over the work required to achieve a given goal is certainly beneficial to all involved. Such process - or implementation path if you will - goes hand in hand with the overarching architecture of the application. Being able to clearly think through the work required to achieve the intended goal while being confident about the completeness of the estimate is one factor through which the accuracy of estimates can be greatly improved.

The primary requirement this imposes upon the application architecture is one of constraints. It requires constraints on the behaviour provided by certain portions of the application such that one is able to confidently reason about the function of a given component. The lack of this characteristic will allow complexity to hide in unforseen nooks and crannies of the application in a way that will screw up even your best estimates.

The implementation of these constraints themselves however require a certain amount of abstract reasoning. The direct implication of this is that a lack of this ability will result in code with poorly defined abstractions and behaviours. The ability to reason about behaviour at appropriate abstraction levels combined with the ability to reflect this through the code is the clearest indication of developer seniority to me.

The implications of the points I laid out in this post revolve around the following aspects:

- Having multiple people providing estimates helps making implicit risks explicit.
- The estimates of junior developers should either be interpreted with a grain of salt or they should be mentored by senior developers supporting them with the appropriate mental models.
- The accuracy of estimates is primarily defined by the transparency or opaqueness of the codebase.

**Estimation multipliers**  
As a closing remark I would like to touch upon the practice of multiplying an estimate by a certain factor. This multiplication would solely disguise the opaqueness which manifests itself in the codebase, and as such becomes an indicator about the quality of the codebase. At the same time I would argue this multiplication factor is inversely correlated to the level of understanding developers have about the work they are doing. The aim should therefore be to reduce the multiplication factor to one while maintaining the accuracy of the final estimates.

Who are we fooling anyway?
