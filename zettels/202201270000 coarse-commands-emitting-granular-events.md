
---
title: "Coarse commands emitting granular events"
slug: "coarse-commands-emitting-granular-events"
date: "2022-01-27"
summary: "In order to allow an event sourced software system to evolve I believe it to be important to issue coarse commands emitting granular events. In this post I explain why."
---

#software-development #dotnet

In response to a previous writing about [the design of event sourced aggregates](https://www.corstianboerman.com/blog/2022-01-20/event-sourced-aggregates) I got a question to consider the situation wherein a single command would dispatch multiple events.

> An example of such situation is described in [the following StackOverflow answer](https://stackoverflow.com/a/21334048/1720761) to the question "**In a correctly modeled CQRS domain, can a single command have multiple events**?".

A major aspect to consider in the design of commands is the way they can be used by consumers of the domain. Too broad and the domain will be useless, too granular and the domain consumer must exert more effort to properly implement the domain. Just right, and everyone should be happy.

## To recap
- **A command**: Used to request an operation. All information should be available, and the command handler may determine whether or not to let the operation occur.
- **An event**: Used to signal an operation is valid, and the internal state may be mutated. Additionally these can be persisted in order to restore the aggregate to a valid state at a later point in time.

When the command is the only object allowed to (or even capable of) instantiating an event, we can be sure that all events in a system have been validated against a certain set of business rules at a given point in the past. The goal of the command then isn't so much to apply these business rules once again, but rather to reflect the previous operation on the aggregate, and to reflect those in the current version of the domain object. It provides a strong separation of concerns, solid conceptual model, which is difficult to break.

## The design of commands and events
In the inevitable situation when the domain needs to be refactored this set up will prove to be beneficial.. Too broad and the domain will be useless, too granular and the domain consumer must exert more effort to properly implement the domain. Just right, and everyone should be happy.

It's this sweet spot which would mostly group a variety of tasks together; take changing user information for example. This could consist of an email, phone and address. These in themselves are unrelated to one another, though relevant for the operation that is being executed. While it makes sense for the command to group these together, not so much for the events.

The events instead would benefit from a more granular approach, which implicitly signals the intention of the operation. As such one would emit three events: `EmailChanged`, `PhoneChanged` and `AddressChanged`. It allows components dependent on domain events to more granularly decide what to act upon, or what not, while keeping the public facade to the domain neatly organized.

## Refactoring commands and events
In the inevitable situation when the domain needs to be refactored this set up will prove to be beneficial.

The general rule of thumb is that commands can be updated anytime without having to worry about anything persisted into the event journal. As long as the domain functions correctly you're good to go.

Since events are the building blocks of the current state of the aggregate we'll need to be slightly more careful with those. As long as existing events can be deserialized into their respective types everything should be good as well. This means that additions can easily be dealt with. Since the event is solely reflecting a single operation (such as the mutation of address details), these events are able to stay relevant over a long time period. This would not be the case when events reflect the exact operation that had been executed. At the same time it allows composition between the command and the emitted events. Whenever the command changes one might decide to drop an event or add another one, thus changing the behaviour of the system without altering events.

This last characteristic is highly beneficial when needing to maintain a system over a longer period of time. One can just discard an event in place whenever it is no longer relevant. The event object in this case still caries the semantic meaning it had carried in the past, even though there is no way one can create a new instance of it. One can ensure backwards compatibility of the event sourced aggregates without having it interfere with the current interaction pattern of the domain object.

## As a general rule of thumb
- Domain commands signal the intention to run a certain operation against the internal state
- Domain commands carry the business rules required to validate a certain operation.
- Domain commands are solely capable of issuing events reflecting the requested operation
- Events are that a certain operation has been successfully validated through a command
- Events carry a semantic meaning of a small (atomic) operation
- Events can be discarded in place when no longer beneficial while still signalling the historical significance of it.

