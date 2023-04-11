---
title: "How to effectively observe the runtime behaviour of your core domain"
slug: "how-to-effectively-observe-the-runtime-behaviour-of-your-core-domain"
date: "2022-07-25"
summary: "Logging statements have no place in the domain. There are multiple better alternatives you should use instead!"
references: 
  - [[202110130000 using-aggregates-in-actor-systems]]
  - [[202208050000 boring-complexity]]
---

#software-development #dotnet

Logging statements have no place in the domain. Within a properly designed software domain one should not be dependent on log statements either. The responsibilities of the domain revolve around state management and the protection of its invariants. Not that of observability.

That is not to say we should compromise on the observable properties of the domain either. There are multiple different ways through which one can gain insight into the functioning of the core domain without having to resort to logging statements. These are the four main observable aspects of a properly designed domain:

| Feature        | Intent                                                                                                                                                                                                            |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unit tests     | Instilling confidence into the stable and correctness of the domain.                                                                                                                                               |
| Commands       | Commands signal the execution of a piece of behaviour. Commands the input arguments to the system.                                                                                                                                                                                                                  |
| Events         | Events signal the successful evaluation of a command, and can be used to confirm the correct functioning of the domain at runtime.                                                                       |
| Exceptions     | Whenever an exception occurs you screwed up. Though I would log exceptions thrown by the domain, I would treat an exception thrown from the domain as a critical bug which should have been caught in unit tests. |
| Result objects | These are the most valuable indications of failure, and should signal the reason of the failure to the calling code. These reasons can be forwarded to the frontend to provide helpful error messages.            |


Except for unit tests, the contents or metadata from all other objects can be reported on from within the infrastructure layer. Just like that logging becomes an infrastructure concern. The main benefit is that logging can be implemented in a consistent and generic manner. This makes it easier to reconstruct the behaviour of the system based on log entries. One is no longer dependent on the inconsistencies that come with logging arbitrary events straight from the domain.

With all information pieced together one should potentially be able to fully reconstruct the behaviour of the system. Each piece of information has their own unique responsibility therein. Commands are the entry point into the domain, and can be logged as such to indicate the start of an operation. The correct functioning of the code is signalled by events. User-errors, or errors coming up from the protection of invariants signalled through result objects. Hard exceptions thrown from the domain are the liability of the developer, and should be fixed as soon as reasonably possible.

One should take care not to log all available information from these object since some of them may contain sensitive information such as PII. Regardless of that, metadata about the operation should usually be enough to recreate a given issue.

Combined together with the unit tests this should enable one to quickly identify the area where the error occurred. To fix an issue it is recommended to first recreate the issue through an unit test, after which one can attempt to alter the logic in a way which makes all tests pass once again. The combination of comprehensive observable properties with test driven development are the preconditions required to be able to respond to and resolve issues at incredibly high pace.

Adhering to this practice keeps the domain clean, while deferring the responsibility of logging to the infrastructure layer. The main benefit is that such approach allows the implementation of logging; and observability in general, in a much more generic manner. It saves ourselves from polluting the code with log statements, while providing more than enough insight into the actual functioning of the code during runtime.

