---
title: "Using aggregates in actor systems"
slug: "using-aggregates-in-actor-systems"
date: "2021-10-13"
summary: ""
---

#software-development

Over the past couple months I have been carrying around this mental block regarding the interoperability between Domain Driven Design (DDD) concepts and the way an actor system is organized. In a way it seems like these are closely related to one another, while at the same time being fairly difficult to reconcile. If the DDD methodology is based on object-oriented programming, and an actor system is mainly based actors and messaging, then how can we combine those two together in a sensible manner?

My initial confusion surrounding these topics initially started out with all the terms which are so closely associated with one another. Usually DDD, event-sourcing, CQRS and also actor systems are mentioned together, and would be used to compose a fully functional system.

Fuelling to the confusion between the application of actors and [aggregates](https://martinfowler.com/bliki/DDD_Aggregate.html) were those similarities and dissimilarities. Both encapsulate their own data, both do not necessarily have influence on their storage medium, though it's only actors which can message other actors within the system, something for which aggregates have to depend on external services and repositories. As such I think it is fair to state that the main difference revolves around its interaction pattern.

With that out of the way, would there still be value for aggregates within an actor systems? Personally I think so. Having this additional constraint could provide valuable freedom in the way a system is designed. Having actors and aggregates explicitly separated enables one to reason on different granularity levels about the functioning of a system. The aggregate then explicitly becomes the place to ensure the internal state remains consistent, while the actors will provide a higher level overview of the functioning of a system as whole.

In the grand scheme of things this separation enables the actors to become merely a transport and persistence layer to the domain model, one which solidifies the interaction patterns in commands and events handled by the actors, delegated to the aggregates.

Though this separation would not necessarily simplify the mental model associated with the system as a whole, it would cut up the mental model in more smaller parts which are easier to handle. Perhaps the most important implication of this is that the cognitive capacity required to work in any given part of the system is significantly reduced, something which I value highly.

Ultimately the state of the actor would then be represented by the aggregate, with the commands mapping to actions on the aggregate. It'd leave the code around actors clean and concise such that the sole focus can be on the messaging contracts, rather than on the code handling the commands.

