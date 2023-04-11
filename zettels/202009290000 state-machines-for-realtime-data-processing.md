---
title: "State Machines for Realtime Data Processing"
slug: "state-machines-for-realtime-data-processing"
date: "2020-09-29"
summary: ""
---

#software-development

After working on realtime data processing logic for hundreds of hours I have identified a few tips and tricks which drastically help simplify the development of these tools. There is an inherent difficulty with regard to realtime data processing, which is that operations need to be executed on single data packets, while incorporating contextual data from the past.

Although realtime data processing initially might look more difficult than batch based processing, this doesn't need to be true. Having a realtime pipeline would allow one to process batches of data, while having a batch based pipeline would not allow realtime processing.

In this post I'll document a few architectural tips and tricks to make realtime data processing look less daunting.


## Software architecture

A thing which keeps surprising me is that more often than not realtime data processing is offered as a feature embedded in tools rather than a programming paradigm. Personally I find it daunting to learn a new tool for something I can more easily build and integrate within my existing software solution.

Over the course of the last five years I have spent hundreds, if not thousands of hours building data processing tools. Every time the common denominator was the fact that it had to be realtime, or if realtime was difficult to achieve, a semi realtime process. Thankfully I wasn't working in finance and therefore speed wasn't the biggest of concerns. Money based computing limitations however have inspired me to try and write somewhat efficient processes, which in turn taught me how to write simple software. Even though simplicity is the key to write great (and maintainable) software, it is almost a necessity for writing realtime data processing algorithms.

Mind you, I'm not talking about the data processing wherein some mathematical operations are applied to a single observation, but the type where statistical operations need to be continuously applied to new data, taking previous data into account. It's this type of data processing where a state machine absolutely shines.

### State machines?

The reason especially a state machine shines is because it transparently shows the transitions between different states. This transparency makes it easier to reason about the whole system, and gives the possibility to identify potential bugs easier.

Over the course of time I have slightly changed the mental model with which I work on state machines. Initially I started out by describing the logic required to reach a certain point. Over time this has evolved in a mechanism whereby I started describing the states in which a (phsyical) system can reside. This lead to unexpected and profound improvements in code quality, and simplicity.

### Describing the state of a system

One of the most impressive state machines I have built is the flight processing logic I developed for use with Skyhop. In essence it accepts a bunch of data points describing the position of aircraft, and returns metadata about the flights such as departure and arrival information.

The following four states describe the critical situation a flight can be in;

- Stationary
- Departing
- Airborne
- Arriving

During a normal flight they should transition in order, however there are also possible deviation, such as an aborted departure where the departing and arriving state quickly follow up on one another.

The essence of these states is that they describe how to deal with new data. Each state has access to the 'context' which describes previous conclusions, and to which each state can append to. The departing state for example checks what type of departure is happening, and redirects the machine to the airborne state to deal with such data, or when the departure is aborted, redirects the machine to the arriving state to track the arrival.

### About the context

The context is one of the most important aspects of the state machine because it enables one to describe the conclusions made by the state machine itself in a transparent way accessible to all individual states. This state is later also communicated to the outside world as the result of the state machine.

This offers a great deal of flexibility. Partly because each individual state does not have to recompute past conclusions, as well as freedom in the way each state reacts to different situations.

Be warned though, because each state can access the context it is possible to create a situation where a state is unable with the data within the context which might possibly halt the whole machine. Though fairly easy to debug when knowing the state the machine stopped in, as well as having the context, this would be a situation undesirable in a production environment.

### Dealing with continuous data

When starting out with state machines I had a machine which processed each data point individually and had a number of states describing how the machine should determine what kind of data point we were processing. This approach essentially takes over some functionality which would be better and easier implemented through the structure of the state machine itself.

Usually a state machine is triggered by specific actions, but in the case of data processing I have found it more beneficial to have a single trigger which responds to newly available data. By convention the states are also responsible to transition to a different state, instead of user action.

### Callbacks

Give we're dealing with realtime data it's only reasonable to asynchronously trigger callbacks to the implementation running this state machine. Incidentally, because the real-world state of the system is described, these callbacks, or usually events in my case, will mostly correspond with the state transitions themselves. When using the realtime state machine to run a batch job I usually access the data through the latest fired event to access the context object and extract all relevant data.

