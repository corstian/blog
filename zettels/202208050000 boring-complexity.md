
---
title: "Boring Complexity"
slug: "boring-complexity"
date: "2022-08-05"
summary: "There are problems in the software industry that have been solved a hundred-thousand times, and will be solved a hundred-thousand more. Rather than re-inventing solutions to these common problems we should be engaging in a discussion about meta-development practices instead."
---

#software-development #philosophy

There are problems in the software industry that have been solved a hundred-thousand times, and will be solved a hundred-thousand more. This rather unproductive and possibly futile manifestation of the "[not invented here syndrome](https://en.wikipedia.org/wiki/Not_invented_here)" seemingly results in an industry-wide stagnation of the development of development-practices. Rather than re-inventing solutions to these common problems we should be engaging in a discussion about meta-development practices instead.

![Slide25.PNG](/uploads/Slide25_4adbfb8ee1.PNG)

## Tension
This itself is what I attempt to bring forth by discussing "boring complexity". Tension exists between the two words making up this phrase due to seemingly impossibility of the combination. Closer examination reveals neither word necessarily contradicts the meaning of the other, and as such the tension should stem from our own associations with the words "boring" and "complexity". Would we rather deal with complexity than be bored? Is (the creation of) complexity a way for us to deal with boredom? I for one am unable to definitively provide a source for this tension. At the same time however I would like to note there might be certain structures which in some perverted way incentivize the creation of complexity far beyond that what is actually required. Be it money, status, or something else.

## To dumb down or collapse
What I find more important however is that we start treating complexity in a way that dumbs it down to the bare necessities. Dumbing stuff down, or developing our own mental models therein is an absolute necessity to be able to tackle additional complexity. Not doing so would lead to stagnation, if not deterioration.

With the unbridled growth of complexity in a way that it is not stabilized on a solid conceptual foundation it is likely that the complexity will collapse in on itself. To extend the metaphor this would result in a dense cluster of "super-complexity" primarily characterized by the difficulty comprehending the full extend of the complexity itself. Forming a complete understanding of the role and impact of this super dense accumulation of complexity would be nearly impossible, for it cannot be overseen by a single person.

This is a more broad problem in an increasingly complex societal structure, though manifests itself rather explicitly in the field of software engineering.

## Not everything needs to be boring
At the same time I must push back against the notion that everything should be dumbed down as much as possible. Attempting to make everything as simple as possible would just be as wasteful as making things unnecessarily complex. For a sufficiently large system however there is a benefit in splitting the relatively stable and mundane parts from the experimental, volatile and complex parts. This way we can focus on creating a super stable core system and screw around with it on the boundaries without fear of breaking the main system. This is what I would consider as being the difference between boring software and exciting software. In such system we can focus on keeping the core as simple as reasonably possible while moving all complexity out to the boundaries.

![Slide26.PNG](/uploads/Slide26_52e1aa0061.PNG)

This way we are able to ensure the exciting aspects create a dependency on the boring system rather than the other way around. This does not necessarily mean that such an external system cannot provide behaviour to the system. It certainly still can, though in a way that the core system does not rely on it. The implication of such thing is that the core system must model the business processes, which can then be automated by these external processes. At the same time the core system can still move independently from these external systems. Though this might break external processes, the main system will keep running at all times.

## Separation of concerns
The distinction between boring and exciting parts of the software is mostly a conceptual one which helps reasoning about where certain behaviour is located. At the same time however the different portions of the system have distinctive properties and characteristics which I will lay out here.

![Slide27.PNG](/uploads/Slide27_0465b46925.PNG)

### Boring aspects
The boring part of the system allows us to focus on all problems which have been solved many times before, and we would be confronted with every single day again if they were not abstracted away. As such the properties and characteristics of this part involve:
- Well defined boundaries
- Well defined solutions to:
    - Structure and data organization
    - Concurrency
    - Consistency
    - Scalability
    - Testability
    - Maintainability
- Stability
- Allows process modelling without externalities
- Allows development task to become routine

Within this part of the system we have two main tools to stay in control of complexity. These are abstractions and processes. Abstractions allow us to keep making sense of behaviour at different levels of granularity, whereas processes exist to maintain some consistency. Consistency without abstractions proves difficult to scale op. Having abstractions without consistency would prevent effective composition of behaviour.

### Exciting aspects
Exciting systems which live outside the boundary of the core system are characterized by the way they are distinctive from the main system. As such they are:
- Contextually decoupled
- Isolated
- Have a well-defined and concise responsibility
- Are independently scalable
- Are independently testable
- Allow the development process to become a creative process

## Development process
There is a rather fundamental difference in what the development process looks like for boring aspects and exciting aspects, which is something we may benefit from. Within the boring aspect the development process is aimed at introducing a process change first and foremost, and therefore the main challenge is ensuring that this change is:

1. What the business wants
2. Compatible with the existing processes

With the predictability of the boring aspects one can much easier develop a stable, well known and predictable process of introducing change in these boring aspects.

This is fundamentally different for the development of an exciting aspect wherein the focus is on automating a portion of the business process. The business process still exists, though outside of the boundary of the excitement. This, by design, provides a clear-cut framework for the expected behaviour of this exciting development in which it may operate. Integration points with the boring aspects are already defined, and within that this component is free to operate. This results in a more clear and well-defined development process, even for highly experimental components. The main challenge during the development of such an external process therefore becomes the technical implementation and the creative process it involves. This allows the implementation of an exciting aspect to be a problem focused creative endeavour.

## Boring complexity
What I mean through "boring complexity" therefore refers to the complexity of the business process as is implemented in a manner that is as technically boring and straightforward as reasonably possible.

It is this simplicity which should preserve the agility of the whole system, even when applied at scale. Ultimately it is the business which should determine the way the system behaves rather than the other way around, and this high-level architectural approach is one of the ways this can be achieved.
