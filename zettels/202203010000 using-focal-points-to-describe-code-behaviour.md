
---
title: "Using focal points to describe code behaviour"
slug: "using-focal-points-to-describe-code-behaviour"
date: "2022-03-01"
summary: "Looking at code through the 'focal points' of the component might clarify a mismatch between the expected behaviour and that what is actually implemented. Organizing code accordingly to these focal points might provide a huge cognitive benefit to maintainers."
---

#software-development #philosophy

Through what I refer to as being a "focal point", I try to clarify the role of a given software component. The focal point itself resemble the conceptual responsibilities of a given component, which one should be able to derive from the name given to a component. It is backwards from the thought process through which conceptuals models such as DDD and event sourcing have emerged, though allows to be a clarifying way to think about the intended way these components should function, what behaviours should be attached, and how they should be used.

Though certain conceptual models fit in exceptionally well within one another (think about DDD, event sourcing and CQRS), I sometimes find that the responsibilities attributed to certain components are incompatible with the composition of the numerous architectural principles that have been applied. One example being that the aggregate is dogmatically regarded as the sole component capable of modifying its internal state in an event-sourced context. The trade-offs one makes to adhere to these dogmatic principles do not seem to fit in with the expected functioning of the software according to the conceptual model employed.

To give a more concrete example about this mismatch I'll try to work out a few focal points for numerous components in a solution modelled according to the DDD principles:

| Component        | Focal point                                           |
| ---------------- | ----------------------------------------------------- |
| Aggregate        | State encapsulation and mutation                      |
| (Domain) Service | Coordination of operations across multiple aggregates |
| Factory          | Instantiation of aggregate or value objects           |
| Repository       | Abstraction over domain persistence concerns          | 

Within a simple domain it absolutely makes sense to express the numerous ways through which an aggregate can be modified as methods on the aggregate itself. In more advanced use-cases one may even return a result-object to detail whether or not the requested operation had succeeded and reflect this result within the UI. All is fine.

As soon as one decides that the domain itself should be event-sourced, these focal points change drastically, as I try to reflect through the table below. It contains components usually associated with DDD and event sourcing, in a way that is implemented in a scalable manner.

| Component | Focal point                                           |
| --------- | ----------------------------------------------------- |
| Aggregate | Encapsulates the state                                |
| Event     | Contains information about how to change the state    |
| Command   | Requests to change the state                          | 
| Service   | Coordination of operations across numerous aggregates |

The most notable change here is that not so much the aggregate decides whether to change the state, but the event decides to do so. Whereas it was previously sufficient enough to have a single operations to both validate the input and validate the state, we must separate these two in order to allow replaying any previously issued events without re-validating the input itself since doing so would make it only a matter of time before the application breaks in unexpected ways.

Depending on how you look at the interaction between the command, aggregate and event objects, you might create a different mental model about how these work with one another. The mental concept of the responsibilities of these objects seems to be fairly universal though, which is represented through the focal point. Since the meaning of these three objects is fairly stable, we might as well construct them in a way that these conceptual responsibilities are actually carried by the objects themselves.

The disconnect between the conceptual responsibilities and the way they are actually implemented starts to occur whenever one moves from the DDD to DDD+ES approach without reconsidering the role of a given component in the grand scheme of things. As with the DDD only approach one was used to modify the state of the aggregate only from within the aggregate itself, and therefore whenever one decides to go with the DDD+ES approach they now end up with a number of command and event handlers in the aggregate. Lingering to the side are the command and event objects solely acting as POCO's for information transfer and persistence.

The scenario I just sketched is disastrous due to the additional cognitive overhead it involves in the maintenance of such approach. The behaviour contained by the aggregate is now centralized into the aggregate, which depending by the number of operations supported might quickly grow into unmaintainable proportions. Not only would it contain a number of command and event handlers which are difficult to navigate, but it is also completely disconnected from the related POCO itself robbing the developer of valuable context while implementing the behaviour.

When organizing the implementation of the behaviour according to the identified focal point of the object, put more concretely the behaviour one might conceptually expect from the role a given component has in the system, we'll end up with a more even distribution of behaviour over a number of classes. Not only is this separation of behaviour more gentle on the cognitive demands it imposes, but it actually provides the developer with all the information it requires to do the job. As for a command this would involve the data associated with the command, as well as logic to validate it against the current state of the aggregate. For an event it would provide the developer with the information attached to it and some logic dictating how it should be applied to the aggregate. The most notable change however is that the sole component being a POCO would be the aggregate, rather than the commands and events. This would make sense from a cognitive point of view since it is easier to remember the contents of a single class, rather than those of a dozen more (thinking about the commands and events).

Personally I deem the added benefits of a reduction of cognitive demands imposed through the code significantly more important than the dogmatic application of design patterns. As long as the application can easily be modified to support new and changing business requirements I'm golden, and the more quickly I can do so, the more compensation I may comfortably demand for my services.
