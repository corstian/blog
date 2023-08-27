---
title: "Architectural improvisations"
slug: "architectural-improvisations"
date: "2023-08-27"
summary: "More often than not we're improvising software architecture as we go. When it is not it is usually over-engineered. What if we adopted an incremental approach to architecture?"
references: 

---

Who isn't experience with the practice? We're building a new software application and we start improvising the overarching architecture of the software we're building. While this works for smallish projects, it is a practice bound to devolve into chaos when we're building larger projects, having to be maintained for years, perhaps decades.

Where I am not going however is advocating for the corporate side of this practice; where architects superimpose a high-level structure from their ivory towers. These projects are bound to devolve into chaos as well, although a different, more bureaucratic kind of chaos.

But then, where is the sweet spot? Ideally we'd have a bunch of overarching principles we can apply in system design. This should come with a deep understanding of the pitfalls of each piece of technology or principle applied. In practice however we can not and should not assume one has this necessary context and insight.

What remains then is an iterative approach to software architecture, but note that is fundamentally different from "just winging it", or "improvising as we go". What sets an iterative approach apart is that there is a clear direction to head into. As we peek further into the future the paths might diverge from the one we're currently on, but as we are not at that intersection yet, it doesn't matter. It's just that by making one step at a time we can continue into the intended direction, and while we might not end up at the point we envisioned, we almost always will end up in a better situation.

But using an iterative approach to software architecture is hard. The reason for this is that certain prior decisions of ours highly impact our flexibility going forward. The trick there is to limit both the number of decisions limiting our future as well as limiting the impact of these decisions. Within software there are plenty of technical patterns for doing so. Our job as software developers is correctly applying these to the present context we are operating in.

One proven trick however is to maintain an "evergreen codebase". It's these repositories of code which may look like they are brand new, even though they have seen several years of active development already. Practically this demands a disciplined and consistent approach to software, which is easiest if you do not keep repeating yourself over and over. While naturally this limits the amount of code there is anyway, the code there is becomes easier to maintain in the face of change. Perhaps an even bolder statement would be that this code would be resilient to temporal changes. This is no easy task though, and requires well-thought out abstractions and shearing points in the software.

**Practical advice**
All software projects are different from one another, but there is some low-hanging fruit which can be picked to get started with the ideal vision of evergreen codebases;

- Do not couple logical rules directly to your API endpoints; not only does this make it difficult to add new API endpoints, but it makes it difficult to change your code as well. An API endpoint can be thought as an interface to access your behaviour. Tear these two concepts apart.
- If you're dealing with a multi-year project, consider using event-sourcing to decouple the information you are handling from the representation of this same information. Event sourcing facilitates radical changes to your data model, which is almost a necessity if you are to follow changes in your business.
- If you are dealing with a DDD domain model, or similarly need to maintain many rules; keep them together, and limit the number of dependencies. Everything else may be infrastructure, but this highly complex core is where the value of your project is.
- When making changes in the way things are done, be sure to apply these changes to prior work as well. It prevents conceptual fractures from arising in the solution. Clean up after yourself as well.
- Work in small iterations. The tighter the feedback cycle, the better. Do not let it loosen up. Deploy often, deploy quickly. Over time this builds trust in your platform as well as your own capabilities. At the same time it provides visibility to small annoyances which build up over time. Clean these up as well, for they accumulate.

But whom am I to tell you what to do. Look around, respond to the context you are operating in, and adapt accordingly.