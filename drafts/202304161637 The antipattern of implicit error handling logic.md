---
title: "Implicit error handling; an antipattern"
slug: "implicit-error-handling-antipattern"
date: "2023-04-16"
summary: ""
references: 

---

#software-development

I for one would assert that the implicit handling of failure modes is an antipattern. The reasons for this assertion have everything to do with handling the complexity which arises in complex software systems.


While I recognize unforeseen conditions may arise in systems, especially so complex systems, I would favour the explicit handling of these, rather than the implicit handling which is often the case. The difference therein can be related to the main purpose of the system.

Systems development, in part, is about the application of constraints to given conditions. Within our full scope of possibilities (that is, within fundamental constraints described by the laws of physics) we are mostly looking to create just the right conditions to achieve our desired goals. It would be fair to state that the set of constraints leading to the preferred outcome is inherently limited, this contrary to the set of constraints leading to undesirable outcomes.

It is for this reasoning that I consider the description of failure modes for a given process to be a futile activity, and thus solemnly focus on approaching the criteria for success. More often than not these success criteria are defined through a process, be it implicit or explicit. Follow the process and the changes of success are reasonably high.

Within a software system the sole implementation of process logic[^1] does not necessarily result in inherently fragile system behaviour. I would argue the contrary for two reasons. First of all this results in more predictable system behaviour. Either the intended outcome is successfully achieved, or it is not. This makes it increasingly difficult to get the system into an inconsistent state. Additionally one can tweak the process to consider edge cases as well. As the main process flow is extended with edge cases we shift the language used to describe error conditions. As language changes, the meaning associated with these conditions changes as well. No longer are we dealing with individual failure modes arising during the main process, but compensatory actions becomes a first class citizen of the system behaviour as well.

This aspect - error handling logic no longer being an implicit addition to our happy paths - has a broad impact on how we treat error modes. Low hanging fruit here is that it makes compensatory actions explicit, something which helps in managing complexity. Keeping this complexity as low as reasonably as possible is perhaps a necessity to prevent otherwise unrelated failure modes from creeping into error handling logic itself. Perhaps more important even is to consider compensatory actions as important as the main system behaviour, leading to increased visibility. This makes it easier to communicate about and collaborate on these aspects of a given system.

Last but not least system maintenance becomes significantly easier and predictable from a technical point of view. As implicit behaviour is no longer hiding in the opaque margins of the system it becomes increasingly uncommon to be startled by complicating factors during system maintenance. Consequentially it becomes easier to accurately estimate the impact of tasks.

Long story short; failure modes of a system should be dealt with in an explicit manner, rather than in the implicit margins of existing behaviour in the system. Doing so will be necessary to reduce the overall complexity, and to ensure the long term stability and predictability of system behaviour. This can be considered absolutely necessary if we are planning to keep a system running for several decades, if not more.


[^1]: I am deliberately not using the word "specification" here. From my perspective the word specification is mostly in a context where the intended outcome of a system is described. A process in that sense is much more specific describing how to achieve the desirable result.