---
title: "Domain integration of external interactions"
slug: "domain-integration-of-external-interactions"
date: "2022-03-17"
summary: "More often than not a software domain is required to deal with external components and service these must integrate with. Here are some considerations around these integration tasks."
---

#software-development

Nowadays a software domain is unlikely to properly function on its own. Within the complex IT landscape there are a huge number of tools which must be integrated in order to provide an adequate solution delivering the expected value.

There are two conceptual methods through which these externalities can be implemented:

1. Have these external integrations push data into the domain
2. Let the domain request data from these externalities

These methods are not mutually exclusive with one another, though they are highly influential in the way an external dependency is integrated into a software domain.

## Context
The way the domain has been designed is highly influential in the way the domain can be constrained. Take for example an event-sourced domain where interaction is defined through commands and the events resulting from the invocation of these commands. Though these could be in place for a non-event sourced domain, it is highly unlikely they are due to the increased complexity. For the context of this post I'll consider the domain to be one that is highly constrained in the way it works, which forces a single interaction pattern upon those working with it. The constraints we're dealing with:

1. Mutation of the domain's state can only be initiated through the invocation of a command against a service or an aggregate.
2. A service can only materialize it's behaviour by invoking commands against other services or aggregates.
3. Commands produce zero or more events
4. Long running processes such as sagas and process managers can solely respond to events
5. The sole way long running processes are able to mutate the state of the domain is by issuing commands.

Practically this leads us to the situation where the sole initiator of a state change from outside the domain can be a command against a service or aggregate. Everything beyond that point is solely a domain concern.

## Decision
In the aforementioned context there are two methods through which one can change the domain;

1. Build a standalone service issuing commands to aggregates
2. Implement a domain service which provides the required behaviour

Herein we must explicitly consider the service to be a domain service including its limitations. That means that a domain service can only be triggered by issuing a command[^1]. A service which is required to be triggered or respond to external calls such as a webhook listener cannot be modeled as a domain service, and instead can only be implemented by having an external component issue a command to the domain.

A service type for which this does not work is one where a component is required to make a call to an external service. These kinds of services are perfectly implemented as part of the (conceptual) domain model. Having a service implemented this way allows one to trigger them with a command from an external component, as well as having them participate in any long-running processes.

And yet the greatest benefit of all is that the integration of these services within the conceptual domain model forces the service to deal with the response from the service in a way the domain model understands. Either this would be an error response, or an event describing the received response (regardless of whether it succeeded or not). Given information retrieved from external services can prove to be crucial for certain decision making processes it is rather important that the data this decision is based upon is properly embedded as part of the domain's state.

Though this would also have been the case when an external service issued commands to domain components, making the services an external concern to the domain introduces the risk of domain behaviour leaking throughout the application[^2].

The risk of having domain logic leak into the codebase is significantly smaller when dealing with external events such as webhooks. Since these events usually contain a piece of information directly relevant to the domain and doesn't need further processing to be made relevant one can usually satisfy with a single domain interaction to embed this information within the domain's state.

[^1]: Practically this would require us to describe the service through an interface within the domain project, while providing a concrete implementation in the infrastructure project. Regardless of this implementation detail it will allow the domain to make calls to such service whenever it makes sense to.
[^2]: Having a leaky domain in place is something to be avoided due to the extreme maintenance costs this ~~might~~ will incur down the line. The resources required to fix such error are in no way comparable with the accumulation of resources required over time to deal with such leak.
