---
title: "A declarative design model for domain driven development with .NET"
slug: "declarative-design-model-for-ddd-with-dotnet"
date: "2023-04-14"
summary: ""
references: 

---

The vocabulary associated with Domain Driven Design, DDD for short, provides a widely understood way to organize code. Among practitioners the conceptual meanings associated to the concepts of "aggregate", "bounded context" and others provide an universal language to understand and communicate structure of software. Though the conceptual meanings of this terminology are widely understood, the technical implementations of these concepts varies wildly among different codebases, cultures and companies. Over the years I have seen my fair share of these projects, and encountered a number of common problems. In this document I propose a rigorously different approach to construct a domain model with the goal of alleviating some of these problems.


The common problems encountered over the years can be summarized by the lack of domain abstractions (e.g. declarative design), and consequentially the excessive layering of abstractions to compensate for this. The impact of these problems were similar across projects as well; excessive code-duplication, ambiguous responsibilities of technical components, vaguely defined behaviour and all second order effects flowing forth from these (i.e. overrunning estimates, poor quality control, poor velocity).

The model outlined in this document is best described as a declarative meta model for the implementation of a domain in C#. In that capacity this model does not constrain the conceptual principles which can be implemented, but governs the way these conceptual principles are technically implemented.

Though C# is primarily an imperative language, there is nothing holding us back from designing a declarative abstraction. The aims of such abstraction are twofold:

1. To allow the composition of behaviour
2. To abstract generic concerns (i.e. storage, logging)

The method outlined in this document is the result of a multi-year iterative process attempting to solve these problems observed in practical use-cases.

Last but not least I want to mention the role of this model in the day-to-day development process. As I see a growing resistance to so-called prescriptive designs, I want to state that although this design shapes a system to a certain extent, it should be flexible enough to expressively describe a domain model. Certainly more expressively so than possible when bogged down by technical debt. These components can be thought about as tools in ones toolbox to implement domain problems, and I hope this document clarifies the impact of these tools slightly.

## High level overview
Each domain component has it's own distinctive role in the overall structure, which will be further discussed in depth later in this document. What is important to understand about the high-level are the following bullet points:

- Changes in the domain can only be initiated through invoking a command or a service
- Invocations of commands or services all evaluate into zero or more events
- Interactions across the boundaries of a bounded context can happen in two ways:
	- A bounded context can use a saga to subscribe to events from another bounded context
	- A command or service within one bounded context can be triggered from another bounded context

![](/uploads/DomainStructure.drawio.svg)

These components are all abstracted and therefore require specific implementations in a concrete domain. The benefit of having all these interactions abstracted is that it allows us to alter generic behaviour at strategic positions in the domain model. Low hanging fruit here is the generalization of persistence logic, but also provides opportunities for the deep and consistent integration of security related behaviour.

## Principles
The structure of this design had been influenced by two goals and corresponding constraints they had placed on the overall design. These goals are:

1. Error handling should not be necessary
2. The domain model should be horizontally scalable

The constraints flowing forth from these goals have far-reaching consequences on the current design.

### Horizontal scalability
Horizontal scalability of a domain model is low-hanging fruit. In my opinion it should not be necessary to refactor a domain model to allow it to scale out. Though - what would generally be considered to be - a significant refactoring effort can easily be justified at a time of (rapid) growth, the resources necessary to do so are always better allocated elsewhere.

#### Consistency
To support horizontal scaling out of the box there are a number of constraints we need to take into consideration. Primarily we will need to ensure that data within the system remains in a consistent state, whether running on a single system or on a cluster. This can mostly be solved by having individual aggregate instances process operations sequentially.

> [!Note]
> Distributed systems are hard. When running a worldwide replicated system we will need to take roughly 300ms latency into consideration to convene with a data centre on the other side of the world. The potential issues arising from these hard physical limits and other complexities are not within the scope of this document. There is a difference again between single-cluster and multi-cluster operations.

#### Network latency
The other problem with distributed applications is the latency arising from round-trips around multiple nodes on the cluster. This is perhaps the single biggest factor slowing down distributed applications, and therefore must be kept to a bare minimum. Designing for this constraint had been one of the biggest challenges; how can complex operations involving multiple aggregates and services be run in a performant manner?

#### Transactions
Not only the latency is a constraining factor, but the method through which multiple operations are coordinated with one another as well. When multiple aggregate instances are modified at once it is undesirable to have a portion of these succeed while the remainder fails. To attempt and prevent this from happening we need distributed transactions. Although these are not completely fail safe, we can manage



---

Rejecting the implicit handling of failure modes in arbitrary domain components results in a situation wherein, in order to deal with failures, these failures must be explicitly modelled as part of the domain itself. This in itself is a favourable characteristic, for the domain itself becomes much more predictable and reliable. Practically this means that one should not write compensatory actions for when part of the operation fails. To prevent the system as a whole from entering an inconsistent state any operation with a failure should be rolled back entirely.

This combined with the second goal of making the domain model natively scalable requires us to implement distributed transactions as well. To prevent excessive locking across the domain, and the associated bottlenecks in information processing, a new evaluation model had to be devised where the intended operations are staged, and only evaluated just in time before side effects should be applied. The lazy evaluation of operations against aggregates allows us to compose complex operations with minimal overhead.

### Error handling
Dealing with errors is perhaps a necessary evil in software development. Especially so in distributed systems where anything may fail at any time for any reason. Grossly generalizing there are two failure modes we need to deal with:

1. Logical failures; an error is thrown because some pre-condition is not met.
2. Infrastructural failures; parts of the system do not respond as expected

The first failure mode is easiest to deal with, and cannot be recovered from as long as some of the contributing factors are not resolved. That is, state, logic or a combination between these. While these are not changed, attempting to re-run a given operation again is futile. Failure might as well be the desired outcome for a given set of arguments. Either way, these sorts of failures can be fairly easily be dealt with throughout the development process.

A more difficult failure mode are those related to infrastructural concerns; that is, the preconditions to evaluate logic and state are not met. This can quite literally happen for any reason.

The recovery model for both types of errors is the same and consists of the following two actions:

1. Ensure the state of the system remains consistent
2. Try to recover from these errors as soon as possible.

Depending on the error mode the second action might prove to be difficult if not outright impossible without intervention.

The composition of commands and services, and their evaluation into events play a crucial role in the error recovery model. If a single command fails to evaluate, no events are applied. If multiple commands to the same aggregate fail, no events are applied. If any operation from a service fails we need a distributed transaction, but then again, no events are applied. If any unanticipated error occurs anywhere, no changes are made to the system.

The only way to deal with errors therefore is to explicitly model them as part of the process. This makes the failure modes of the application explicit, and makes behaviour more predictable.

### Transactions
Distributed transactions are a pain to do right, and even then still have failure modes which might occur in certain edge-cases. At the same time it is considered too costly not to have transactions due to the increased complexity involved in building complex behaviour without.

The consideration of having to run distributed transactions significantly impacted the way operations are evaluated. As to prevent extensive locking of aggregates it had been decided that the number of consecutive round trips should be reduced as much as possible. This is achieved by having operations return their intended state changes without evaluating them already. Only after all intended side-effects of an operation are collected, they are concurrently validated with all aggregates involved. If any of these evaluation would fail, the whole operation would be rolled back. This allows the use of distributed transactions which apply complex operations in a single, though concurrent round-trip among all involved.

Because the desired characteristics of the distributed commit protocol are highly dependent on the application in question, the distributed commit protocol is transparently pluggable.

### Concurrency and scalability
The concurrent model of evaluation and scalability characteristics of an application are closely related to one another. Different components within the domain relate differently to concurrent models of evaluation.

An aggregate, having to protect state, can therefore only evaluate operations synchronously and sequentially. By preventing extensive locking and just-in-time evaluation of commands we can maximize throughput as much as possible. Because of this execution model, an aggregate instance should be unique across a cluster. 

Services, contrary, are not subject to limitations on concurrency. Because services are not reliant on state, any number of a service can be concurrently evaluated, with the only limitation being external dependencies and the aggregates on which the resulting state changes should be applied.

Sagas are different altogether. Once an event is confirmed, any saga built to the event type is evaluated to determine how to continue the operation. Due to the stateless nature of sagas it does not matter on which node in a cluster these are evaluated. For speed it is therefore convenient to evaluate these on the same node where the event had been confirmed. This provides the additional benefit that it is possible to provide the saga with access to the current aggregates state. This further reduces the need for the use of process managers.

When the domain model runs within a cluster the only communication happening across nodes are 

This results in an execution model which works on single node instances (locally), and easily scales out across a cluster.


## Usage
### Aggregate
The aggregate can easily be considered the foundational cornerstone for any DDD implementation. Many examples will show a (simplistic) aggregate resembling something similar to the following:

```csharp
public class User {
	public string FirstName { get; private set; }
	public string LastName { get; private set; }

	public void Rename(string firstName, string lastName) {
		if (string.IsNullOrWhiteSpace(firstName) || string.isNullOrWhiteSpace(lastName))
			throw new Exception("Name is required");

		FirstName = firstName;
		LastName = lastName;
	}
}
```

The challenging aspect of with this code is that it has a number of implicit responsibilities. The problem in this case with implicit responsibilities is that these cannot easily be generalized across the domain. To break these down:

- The aggregate holds state
- The aggregate contains the `Rename` method; which can be thought about as being a command
- The `Rename` method does input validation
- The `Rename` method provides feedback about input validation
- The `Rename` method changes the state of the aggregate

> [!Warning]
> Aside from these implicit responsibilities it is worth noting that refactoring these (non generic) aspects across a whole domain would be a herculean task. I have seen a project where no (consistent) feedback about evaluation failures could be provided because that was not considered in the initial design. The workaround involved explicitly designing event flows to pass on these validation errors asynchronously to an event listener on the client which initiated said call in the first place.

To make these aspects explicit we're splitting them up in the following three conceptual components:

- Aggregates; responsible for holding state
- Commands; responsible for argument validation
- Events; responsible for making aggregate state changes

Furthermore these components themselves do only contain state, whereas a handler equivalent will  handle the logic associated with these components. With this explicit definition of responsibilities an aggregate will look as follows:

```csharp
/* Note that IAggregate is merely a marker interface */
public record User : IAggregate {
	public string FirstName { get; init; }
	public string LastName { get; init; }
}

public interface IAggregateHandler
{
	public Task<Result<IEvent[]>> Evaluate(
		IAggregate aggregate, 
		params ICommand[] commands);
		
	public void Apply(
		ref IAggregate aggregate, 
		params IEvent[] events);
}
```

The `IAggregateHandler` interface in the code-block above is the first generic extension point available at our disposal. A concrete implementation for such handler can be provided through a dependency container such that the core domain does not need to know about the specifics about how it deals with aggregates, and how they related to commands and events. This same approach to abstract the concrete relationships between objects will be applied to most other domain objects as well.

### Events
The conceptual role events fulfil is comparable with what Eric Evans referred to as being assertions. The intent of these is to make side-effects explicit, and therefore predictable. This is done by ensuring commands and services can only introduce side-effects by issuing events. At the same time the use of events facilitates the straightforward yet strategically important implementation of event-sourcing techniques.

Events can be represented as follows:

```csharp
public record NameChanged(
	Guid AggregateId,
	string FirstName,
	string LastName
) : IEvent;

internal class NameChangedHandler<User, NameChanged> {
	public Profile Apply(IEventHandlerContext<User> context, NameChanged @event)
		=> context.Aggregate with {
			FirstName = @event.FirstName,
			LastName = @event.LastName
		};
}
```

It must be mentioned that even though the event is able to access the aggregate, it does still only live on the boundary of the aggregate. Even though it is able to access the aggregate itself, it does not have unconditional access to the internals of the aggregate. If the implementation of the aggregate does obscure certain implementation details this will not be visible to the event either.

### Operations
All operations will return either  zero or more events, or result in an error. Both commands as well as services are considered as operations, even though their semantics are different. Their semantics are discussed more in depth here;

#### Commands
The sole way to initiate a change on an aggregates state is through the command. The command is responsible for validating whether the intended change is valid given the current aggregates state.

The semantics are similar to those of the event, but the capabilities of a command are wider, as in that it cannot solely access an aggregate through the provided context, but can also stage events.

The command and its handler are represented in code as follows:

```csharp
public record ChangeName(
	Guid AggregateId,
	string FirstName,
	string LastName
) : ICommand;

internal class ChangeNameHandler : ICommandHandler<User, ChangeName> {
	public Result Evaluate(ICommandHandlerContext<User> context, ChangeName command)
	{
		if (string.IsNullOrWhiteSpace(firstName) || string.isNullOrWhiteSpace(lastName))
			return Result.Fail("Name should be set");
		
		context.StageEvent(new NameChanged(
			command.AggregateId,
			command.FirstName,
			command.LastName));
		
		return Result.Ok();
	}
}
```

The staged events are only applied to the aggregate after the command is successfully evaluated. This behaviour is dependent on the concrete implementation of a piece of infrastructural glue, which will be discussed more in-depth later on.

#### Services
The self similarity in the way domain components are handled also extends to services. As such services are similar to commands. Distinctive to services is that they can not stage events, but instead must invoke commands to have any meaningful impact on an aggregate.

The distinct benefit of a service is that it lives outside of the boundaries of an aggregate and therefore is not subjected to the constraints this boundary represents. The service can therefore:

- Make asynchronous requests to external resources
- Stage commands for future evaluation
- Invoke services for immediate evaluation

Since services stage commands for future evaluation, each service invocation, if successful, adds further commands to be evaluated upon completion. This method of deferred evaluation provides beneficial and valuable characteristics for the future composition of behaviour.

The way a service is implemented in this conceptual model is distinct from what is generally considered a service. The semantics are similar to those of a command.

```csharp
public record SynchronizeNameService(
	Guid UserId 
) : IService;

public class SynchronizeNameServiceHandler : IServiceHandler<SynchronizeNameService> {
	private readonly IPaymentGatewayAPI _paymentGateway;
	
	public SynchronizeNameServiceHandler(IPaymentGatewayAPI paymentGateway) {
		_paymentGateway = paymentGateway;
	}
	
	public async Task<Result> Handle(IServiceHandlerContext context, SynchronizeNameService service)
	{
		var account = await _paymentGateway.GetAccount(service.UserId);

		context.StageCommand(new ChangeName(
			service.UserId,
			account.FirstName,
			accoutn.LastName
		));

		return Result.Ok();
	}
}
```

Because a service is able to use invoke external actions it is the responsibility of developers to ensure that external systems stay up-to-date with the current internal state or vice-versa. Due to the volatility of external resources there is no guarantee that the invocation of an external action is going to succeed. Common methods this uncertainty can be dealt with are:

- Fail the whole operation
- Explicitly model failure as part of the model
- Let external resources push updates to the present system

### Sagas
The saga is a component which takes a confirmed event and translates this in one or more operations. This proves to be especially beneficial for integrating multiple bounded contexts with one another without tightly coupling these to one another.

> [!Info]
> The difference between a saga and a process manager is commonly misunderstood, in part due to different meanings used by different tools and frameworks. In this case the saga is a component which is stateless, this in contrast to the process manager. If state is necessary to keep track of multiple incoming events, an aggregate can easily be purposed to fulfil this requirement.


