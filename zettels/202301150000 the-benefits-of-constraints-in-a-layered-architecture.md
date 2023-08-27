---
title: "The benefits of constraints in a layered architecture"
slug: "the-benefits-of-constraints-in-a-layered-architecture"
date: "2023-01-15"
summary: "I hypothesize that a codebase which can be navigated by logical rules provides greater accessibility than one that must be navigated through memorization. This is one of the reasons why small teams could benefit from a layered architecture."
references: 
  - '[[202201200000 event-sourced-aggregates]]'
  - '[[202206240000 how-complex-software-impacts-your-cognitive-abilities]]'
  - '[[202208050000 boring-complexity]]'
---

#software-development

In the past while I have seen more attention being attracted towards an architectural pattern considered as vertical slices. This pattern in itself is a counterreaction to the patterns generally known as comprising a layered/clean/onion architecture. The reasoning for favouring the vertical slices is that having the ability to specifically choose the stack for every new problem provides a certain sense of freedom and specificity in the way a problem is tackled. At the same time it is pointed out - and this is certainly a valid point - that it prevents dogmatism from prevailing and colouring the chosen solution for a given problem.

Irregardless of the validity of the arguments used in the vertical slices vs layered architecture debate, I personally think the overwhelming majority of applications would still benefit greatly from a properly layered architecture.

Vertical slices certainly have their benefit in large codebases maintained by large teams, but I would categorize these patterns alongside microservice approaches. Just like microservices they are great applied in certain contexts, though also have a great potential to act as a productivity sinks when improperly applied.

With this in mind I would assert that it is more beneficial for small teams to work according to a more uniform and coherent approach. This could thus include elements from a layered architecture. The underlying mechanism improving productivity, I would assert, is that a constraint is placed on the number of ways a certain goal can be achieved. It's such constraint which in itself goes against the philosophy of vertical slices. For the remainder of this post I will explicitly focus on the way through which constraints help improve productivity.

> As a rule of thumb I hypothesize (properly) layered architectures to be working great in codebases up to roughly 100k LoC, which happens to be a rather practical limit to the size at which a single person can still understand the extend of the solution. Larger and it becomes increasingly challenging for a single person to understand and maintain the software. It is for codebases larger than this limit where I imagine vertical slices to become increasingly valuable.  

## The temporal progression from CRUD to DDD
Through time we can see a progression in the sorts of responsibilities applications take on. The time that applications merely took on the responsibilities of a document drawer seems to be over. In the application architecture this seems to be signified by the well-known architecture of CRUD applications. Nowadays this architecture type is rapidly being commoditized due to the inception of no-code tools allowing laypeople to quickly create these for themselves. Regarding the development of custom applications we see that these quickly take on more responsibilities beyond simply storing and altering records. Instead we see that these take on the previously human responsibility of ensuring the records contain the correct information, and these are consistent with one another. This development in itself seems to signify the progression from CRUD applications to applications architectures constructed according to DDD methodologies, which is the de-facto standard for understanding and maintaining the cohesion of validation logic contained on records of information.

Even in the time when CRUD applications were still the de-facto standard, there was already opposition to generalizing CRUD behaviour, even though at the current time such thing seems to be totally ridiculous. Arguments I had heard back then was that a generalization would not allow specific performance improvements, or that it would not allow any potential and specific behaviour to be attached to a certain operation. All these problems have been solved over time as evident by the plethora of no-code tools enabling nearly everyone to achieve these things. With the current tool-chains we possess these things also have become trivially simple. And yet I would argue these no-code tools would not have existed without first generalizing this behaviour in code.

From this perspective it seems to be only a matter of time before the archetypical DDD way of structuring ~~code~~ behaviour is to be commoditized in a no-code tool, and yet before we can get there I believe we should first write the abstractions necessary to consistently and coherently construct a domain model in code.

## The logical structure of domain code
Most application nowadays depend on a common set of technical requirements. Among these are:

- Users or machines should be authorized and authenticated
- Data and mutations thereto should be persisted
- Data operations should be checked against one or more rules verifying whether such operation is allowed to proceed
- External applications should be able to interact with the data as well (i.e. receive and initiate mutations)

In a data-centric (DDD structured) environment we'll see the following granularity in the way data is structured:

- Values; represent shards of data repeated throughout the system (e.g. a physical address)
- Entities; represents a (part of) conceptual object
- Aggregate; represents a whole conceptual object, made up of entities
- Services; represent operations among multiple aggregates
- Bounded contexts; represent the scope of a single contextual domain (e.g. the difference between financial and HR departments)

Though these aspects can access the concepts contained by them, none of these can access anything outside of their scope. While the granularity dictates the scope of the data contained within, the limitation in the way information can be accessed dictates the way behaviour is organized. Because of this rather fundamental yet philosophical constraint in the way both data and behaviour are organized one can predict where a certain piece of behaviour can be found without knowing the exact nature of this behaviour itself. This allows one to reason through the behaviour of the system using a few rudimentary guidelines, without having to know the intimate details of the functioning of the system.

The structure the DDD methodology provides is one example of how a rudimentary constraint allows one to move with more creative freedom through an existing structure. In this case it allows the brain to reason through a limited set of rules to find ones way, rather than having to remember a set of seemingly incoherent details.

> I hypothesize that a codebase which can be navigated by logical rules provides greater accessibility than one that must be navigated through memorization. The reasoning for this is that those having difficulty memorizing information can rely on logical deduction, while those easily memorizing information will memorize anyway, irregardless of whether this information can be logically deducted.

## Proper layering of behaviour
When implementing an application, and perhaps even more so when extending or maintaining an existing application, all I would care about is the way behaviour is implemented. What I explicitly do not care about are the technical details facilitating this behaviour. This, in my humble opinion, is something which should be considered during the initial design of an application.

Given all that we care about is the conceptual behaviour that is implemented we should structure the application as such. An antipattern therein is to have to dig through several abstraction layers before a certain piece of behaviour is implemented. Explicitly this means that one should not be required to implement a model, a viewmodel, database persistence logic and what not might be required to get a certain feature to work. Instead it should be expected that these technical dependencies are readily available when new behaviour is implemented. The implementation process should be as simple as;

1. The conceptual implementation of new behaviour
2. Wherever necessary; exposing the new behaviour to the outside world

Technically this can be achieved by generalizing the interaction model through which one addresses the domain, and using these as hooks to wire up generic behaviour associated with these operations. I have written more about how this works in various different posts. See the references next to this post for these links.

## Technical translation of conceptual models
If properly structured, there is an immense benefit to having a layered architecture in the implementation of conceptual models from the real world. This benefit is that the cognitive complexity associated with a translation of these conceptual models into their technical representation is kept to a minimum. One factor contributing thereto is the coherency and consistency with which conceptual models can be translated into technical models. There is a small ruleset according to which one can easily reason back and forth between both modes.

It is here once again that constraints in the way one does things leads to more creative freedom for there is a certain consistency in the way these problems are tackled. The translation of conceptual models no longer needs to be a conscious effort, but instead becomes a behavioural process through which one can think about the conceptual implication rather than the technical ones.

## Deviating from the generalized structure
Then whenever such generalized infrastructure for the implementation of domain behaviour is available, I would argue that at best it is a waste of time to re-implement such technical foundation, while at worst it causes grave inconsistencies in a single platform between the way data and behaviour is dealt with. Generally it is possible to create such generalization in a consistent and performant manner, whereas specific requirements might require one to deviate from this. Of course I am not opposing such case specific deviations, as long as they fit within a general philosophy of systems design. Within the DDD world we'd mostly be looking at projections for such operations, where a specialized data store is built based on data from a centralized data store. Such approach at least ensures predictability across various portions of the application.
