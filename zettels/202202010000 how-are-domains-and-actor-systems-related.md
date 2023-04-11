
---
title: "How are domains and actor systems related?"
slug: "how-are-domains-and-actor-systems-related"
date: "2022-02-01"
summary: "A software domain and an actor system both solve specific problems, and yet they complement each other in the creation of scalable and yet testable system. Some thoughts about how they interact."
---

#software-development

Domain driven design and actor systems are two things which work together in a great manner. They complement one another if you will.

The fundamental problems which are being solved by both pieces of technology are rather different. Where a domain primarily focuses on the consistency and integrity of data, an actor systems primary responsibility is to ensure scalability and availability of information. In relation to the domain an actor system is merely a piece of convenient infrastructure.

Viewing the actor system as exactly that, a piece of infrastructure, should have profound impact on the way we treat the actor system. Rather than loading it to the brim with functionality we can develop individual components in isolation, and host them through actors instead. The also applies to a domain. When the domain is properly decoupled from the actor system one can use the actor system as a piece of infrastructure responsible for the scaling and availability of the domain.

This way we achieve a clear separation of responsibilities; the actor system solves our scaling, availability and persistence problems, while the domain holds the business logic itself. Since there are hardly situations where one needs to focus both on the actor system and the domain logic at the same time this segregation will prove to be beneficial in daily maintenance tasks on a given software system.

## About testability
This separation will quickly prove to be beneficial for testing the system. While writing unit tests one no longer has to mock the actor system to test behaviour contained therein, nor has one to deal with domain logic to test the actor system. The resulting situation is one where tests are specific to a smaller part of the codebase, and the result is that in case of a failure, the underlying cause(s) can quickly be located and fixed.
