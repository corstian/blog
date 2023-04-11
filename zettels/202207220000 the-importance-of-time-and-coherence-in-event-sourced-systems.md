
---
title: "The importance of time and coherence in event sourced systems"
slug: "the-importance-of-time-and-coherence-in-event-sourced-systems"
date: "2022-07-22"
summary: "To make a system more maintainable its temporal characteristics should mimic the way humans perceive the passing of time. Coherence should bind the past and the future through the present, whereas continuity should explain the present by the past."
---

#software-development #psychology #philosophy

At this point in time we are not yet convinced that software has a conscious and coherent experience of the passing of time. While a given system is still subjected by the passing of time, it may conceptually intertwine concepts of the past, the present and the future.

While on certain abstraction levels it may be fine to disregard the coherence between temporal relationships of behaviour, maintenance thereof would prove to be challenging for humans. Unlike systems, the perception of time by humans is influenced by the characteristics of the past, and the future, bound by an intangible moment known and experienced as the present. The past is defined by its immutability, whereas the future is (partially) unknowable. It is only the present in which the past can be defined, and the future can be influenced. As abstractions in logical systems can be freely defined (again; unlike humans), these systems have more freedom in the conceptual interpretation of time than humans do.

To make a system more maintainable, I would argue, we should model the temporal characteristics to mimic the way humans perceive the passing of time. We may characterize this human experience based by two aspects. First there is a coherent experience of time which binds the past and the future through the present. A second and dependent aspect is the existence of continuity, whereas current events are impacted by the past, but will also influence future events. There must be a logical succession of events explaining the present.

Approaching software design and taking the human perception of time into account we will end up with a number of recommendations which should make it easier for humans to reason about the behaviour of said software. By modelling software products in a manner compatible with our own perception we'll remove some of the cognitive burdens that naturally come with alternate mental models. I would argue the passing of time is such a fundamental aspect to the human experience that it is not only difficult to come up with an alternative approach, but if one exists it might be outright impossible to reason through its functioning in a productive manner. For this reason I believe software systems should attempt to incorporate this perspective into their functioning.

## The solution of event-sourcing
Perhaps I'm kicking in an open door by referring to the technical pattern of event-sourcing in the context of temporal relationships. Depending on the exact implementation, event sourcing already attempts to model the temporal relationship between the present state and past operations. Ultimately the current state can be considered the sum of all past operations.

An amazing capability event-sourcing provides is the ability to re-assess historic events to alter the current state. In doing so we are not altering the events themselves - after all the past cannot be changed - but changing the impact of these prior events on the current situation. This is hugely beneficial when evolving an existing system. Without event-sourcing one could only assess the current state of the system, without the rich historical context that led us to the present moment. It will prove to be the difference between a reactive and pro-active development approach.

This is analogous to the way humans process the past, and are able to re-assess it. Experiences themselves cannot be invalidated, whether good or bad, but the bias they caused in the present interpretation of reality can be re-assessed. Without this ability one would eventually develop coping mechanisms ineffective for dealing with the present. Some of which might prove to be outright destructive. Regular re-assessments are required to maintain ones mental well-being over time.

Just like humans suffer when they cannot depend on their experiences, a system unable to evaluate past decisions is bound to collapse under their own weight far more quickly than a system that would.

When building a system that is supposed to last for decades (perhaps centuries?) event-sourcing is a necessity, for otherwise the system will devolve into chaos, and its developers will become depressed.

## The past as immutable fact
Just like humans cannot change the past, I wouldn't recommend software systems to alter the past either. The sole window in which we may alter the future is the present, and the opportunities we have in this moment are defined by the context and decisions made in the past.

Ignoring this coherence in the flow of time would break the continuity aspect we started with. In the hypothetical situation we were to break it we could choose to deal with this in a number of different ways. The first one is to propagate the change and alter later events. The danger of this approach is that we might end up with a time-travellers paradox, where changes are reversed as they become incompatible with the altered timeline. This is not even considering whether such changes must adhere to current or historically held invariants, which itself is a rather difficult question. Altogether it would generally be an undesirable property of a software system to destroy information based on a change introduced in the timeline. An alternative approach is to allow changes of the timeline without re-evaluating subsequent changes. Though the implications of this approach are less severe than the time-travel paradox, we are now unable to protect our invariants. Allowing ourselves to break invariants would increase the rate at which the system deteriorates since one can no longer be sure about the exact contents of data contained within. 

## Dealing with the past in the present
That is not to say it is easy to reason about the contents of the data contained within the system. In fact, it might be the most difficult aspect of keeping a given system up and running. Invariants are bound to change over time, which result in certain ambiguity about the data contained therein.

There is yet another analogy to human nature to be found there. Generally humans seem to be able to exclusively judge based on their current understanding and context, which is recognizable in the form of shame. (Thankfully the existence of shame is a tangible result of personal growth.) It makes it all the more ironic that historic actions are often attempted to be framed in their historic contexts, more often than not solely leading to apologetic behaviour. This itself is completely useless as not only can the past not be altered, but it is also the current context that we need to deal with, regardless of any past decisions.

As such keeping historic invariants around is fairly useless, since the sole thing we are to deal with is the present. If one ever needs to consider these for archaeological reasons it can generally be considered to exist within the version control system. Generally there is no reason to deal with these, except when it is required to re-assess the impact historic events ought to have on the present.

Ultimately we do not deal with past behaviour, but a description (or consequence) of it instead. A strict separation ought to exist between our reasons for behaving and the impact of said behaviour on the present. It is our interpretation of the significance of past events which might change, but we cannot change the decision making that happened beforehand.

## Evolution
These considerations leaves us with a rather strict way through which we are able to evolve an event-sourced system to adapt to new requirements. Generally the invariants can be changed all the time, as they only impact the present decision making process. This is much different for events. Given events reflect the result of historic operations the information contained within should be preserved. What we may change however is the way events are interpreted in the present. Generally we may be able to do so through altering the way events are replayed and applied.

From a practical point of view it is therefore important that a single event has the smallest possible footprint. This allows us to compose multiple events to make up more complex operations. It's not so much the events which hold the invariants we're trying to protect, but these complex operations. It's the events merely representing a result of the evaluation of these invariants, and as such they are merely descriptive.

This allows us to freely alter any invariants to support the present use-case, while also allowing to discard events in-place when they are no longer necessary. Leaving them around signifies the historic significance of them, and still allows one to re-interpret their impact on the current system as needed. This approach of dealing with events provides a technical solution to deal with the difference between the past and the present, based on which we might influence the future. All while maintaining our ability to reason about the current and future states of the system.
