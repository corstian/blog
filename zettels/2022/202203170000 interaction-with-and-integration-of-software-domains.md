---
title: "Interaction with and integration of software domains"
slug: "interaction-with-and-integration-of-software-domains"
date: "2022-03-17"
summary: "A software domain has the privilege to live in total isolation from any other component in the codebase, though it must be consumed by other components. By generalizing the interaction pattern to the domain one can greatly simplify these integration points."
references: 
---

#software-development

The interaction pattern of the domain, or rather the way you deal and interact with the domain, is defined by a number of different aspects. Each of these aspects can be simplified to a design decision which provides a number of benefits and tradeoffs. The table below shows a number of design decisions one might make:

| Aspect                                                        | Benefit                                                                              | Tradeoff                                                  |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| Event sourcing                                                | The state of the domain is disconnected from the operations executed on the domain   | Increased complexity                                      |
| CQRS                                                          | The domain does not need to produce a queryable representation of the data contained | Additional overhead in maintenance                        |
| Method based interaction                                      | Easier to comprehend; one less abstraction to deal with                              | Difficult to generalize                                   |
| Object (message) based interaction                            | Easy to generalize                                                                   | Involves an additional abstraction to understand          |
| Support for long running processes (sagas / process managers) | Long running processes are explicitly represented in the codebase                    | Additional temporal complexity to take into account       |
| Level of constraint (by design)                               | Having the inability to work around the domain                                       | Limited flexibility regarding how a domain is implemented |

All of these design decisions together uniquely define how one must interact with the domain. Despite these aspects the fundamental architectural consideration foundational to a software domain still hold, and these aspects are simply extensions to that model.

## In a broader sense
The domain should be the most isolated part of the codebase. It preferably has no dependencies on any other code. At the same time it would be the component having most dependents. The domain is the part where all the decision making happens, and therefore any business decision made by or through the code should be reflected in the state of the domain.

> Even when an external component is used to make a decision (think about machine learning applications), a certain amount of information about this should be reflected in the domain for tracing and accountability. In a case where machine learning is involved one might need to store the timestamp, as well as input parameters and the output value(s) for accountability.

However valuable a domain model can be on its own, it's virtually worthless if no one is able to interact with it. Therefore one goes to great lengths to make the domain model accessible to outside components. Whether that is directly through an interface, or through yet another abstraction in the shape of an API.

The goal of the API itself is to create an abstraction between the domain model and the implementing client to allow a one to many relationship between those two components. It allows the client, wherever it is implemented, to become a domain consumer just as much as any other component in the codebase. Thinking about a given front-end as yet another domain consumer makes the intended design of the API surface more clear; it should reflect the domain model in order to be able to use the same mental model to both work the backend as well as the frontend.

This is rather convenient as this allows us to develop the API in a manner which makes it a flimsy addition on top of the domain itself. It becomes a transparent mapper between the interfaces of the API and those of the domain. Modelling an API this way perhaps involves the least amount of code reasonably possible, since it allows a level of generalization not otherwise possible.

## Domain interaction
There are two main approaches to interact with a domain model;

1. Through method invocations
2. Through object (message) passing

Though either method is reasonably scalable in the technical sense, I found them to have their own distinctive pros and cons. Interacting with the domain through methods would be a more straightforward and easier to implement approach for less experienced developers, since that is as close to the language as can be. Message passing in this regard introduces another layer of abstraction which must be understood first. Despite that I find message passing to be easier to generalize and therefore becomes much easier to work with from a cognitive point of view.

> When using messages to interact with a domain model it is much easier to define a single contract describing the type of a message and what it can do, and have a single handler process all those messages. The sweeping generalizations it allows one to make are a delight to work with.

## Requirements
The sole way this can be made to work however is to ensure that the information the domain returns is enough to create a reactive interface with. There are generally two pieces of information the domain is able to independently return:

1. Events (in case of an event-sourced domain)
2. The current state (preferably through snapshots in order not to create external dependencies to internal state)

Based on this data one generally would be able to design an API which functions correctly without over-fetching any data. The reasoning behind this approach is simple; whenever an aggregate is modified it is just a small effort to return data reflecting the operation that had just been applied to the data. Since the scope of the mutation is limited to the boundaries of the current aggregate, or in case of a service to the combined boundaries of the aggregates the service touches, we can be sure there are no direct side effects from the operation that had just been executed.

This comes with a footnote, and that is the way any long-running processes are modeled. Within the domain there may be a number of components which trigger based on the events that had just been emitted. Since the general rule of thumb is that subscriptions on domain processes are only to be used for long-running processes in the temporal sense of the concept one can be sure the the resulting response is the most immediate response one can expect. To receive updates on any long-running process one must explicitly subscribe to real-time updates on the appropriate channels.

This however does not mean that events cannot be used to chain a number of domain operations after one another. A service is perfectly suited to issue a command to an aggregate, await its result, forward it to another service also awaiting its result, to finish off with persisting the latest received response in an aggregate through issuing a command. The specific response that event might issue would involve the events generated by two commands in addition to those abstracted away through the service called as the second operation. This would provide the domain client with full visibility about the operation without depending on an asynchronous communication protocol. 

## Domain consumers
Additionally such interaction pattern would provide a straightforward way to integrate the current state of the domain model within a domain consumer. It is up to the domain consumer to determine whether it wants to keep it's own internal state based on the events it receives or (more likely) just update its internal state with the snapshots it receives from the backend.

For this latter method one solely requires a normalized cache wherein documents are stored based on their type and identifier. It allows the consumer to integrate any state change that occured and update its internal state based on the information it receives without the need to issue further queries to the backend. The sole queries which are required are those to fetch the initial state. Anything after can be updated through real-time communication channels and the results from the operations which are executed against the domain.
