---
title: "202304191706"
slug: "202304191706"
date: "2023-04-19"
summary: ""
references: 

---

#software-development 

The `Whaally.Domain` library provides a declarative meta-model suitable for the construction of a software domain in C#. This post dives into the technical implementation details of this library.


The foundation of this project revolves around a number of loosely coupled interfaces. The interface is highly suggestive of how these different components should be pieced together, but provides complete flexibility in the specifics of this implementation. The basic interaction pattern with a domain implemented using this library can be visualized as follows:

![DomainStructure.drawio](DomainStructure.drawio.svg)

The sole way the user-developer can interact with the model is through invoking commands and/or services. Unique of this model is that both of these operations can locally be reduced to a set of commands describing the intended state changes to the aggregates involved in the operation. Through the concurrent evaluation of these commands against their respective aggregates all operations eventually result in a number of assertions reflected through a set of events describing the intended state-changes as a result of said operation.

## Extensibility
The `Whaally.Domain` library had been written in a way that generic domain behaviour can be strategically modified by providing a custom implementation of a certain interface. To achieve this the library makes extensive use of dependency injection techniques to retrieve current interface implementations from a dependency container.

### Handlers
As a general rule of thumb each domain component (i.e. aggregates etc) also has a handler providing concrete behaviour for that component.

Please note that for most of the interfaces described below generic versions exist which takes concrete implementations of these interfaces as generic type property to rely on the type checking capabilities as much as reasonably possible.

#### Aggregate
The `IAggregate` is a marker interface used to provide a generic constraint for other domain components requiring an aggregate type to be provided. At the time of writing this interface dictates the addition of an Id field of the Guid type to be able to identify which instance we're working with.

> [!Info]
> If necessary we might as well look into a more flexible way to provide an identity and to allow for different data-types. The benefits an explicitly defined identity provides have been considered to be worth this trade-off for now.

The `IAggregate` and any concrete aggregate implemented using this interface are only intended to be plain Data Transfer Objects, DTOs for short. They only need to provide information about the current structure of the data contained within the aggregate. In order to define how this data can be changed we are working with a `IAggregateHandler` interface which defines an `Evaluate` and `Apply` method. These two methods define the interaction between the aggregate and related commands/events.

```csharp
public interface IAggregateHandler
{
	public Task<Result<IEvent[]>> Evaluate(IAggregate aggregate, params ICommand[] commands);
	public void Apply(ref IAggregate aggregate, params IEvent[] events);
}
```

The `IAggregateHandler` interface and any implementations can be thought about as being the glue between the aggregate and any commands/events. The relationship between these objects as defined in an implementation of the `IAggregateHandler` generally remains consistent within a single project.

#### Command / Event
The structure of commands and events are similar to one another. Again we see that the `ICommand` and `IEvent` interfaces function as a marker having only a reference to the aggregates id as required property. Behaviour for these objects is implemented through implementing the `ICommandHandler` and `IEventHandler` interfaces.

```csharp
public interface ICommandHandler
{
	public Result Evaluate(ICommandHandlerContext<IAggregate> context, ICommand command);
}

public interface IEventHandler
{
	public TAggregate Apply<TAggregate>(IEventHandlerContext<TAggregate> context, IEvent @event)
		where TAggregate : class, IAggregate;
}
```

Where these handler objects meaningfully differ from the `IAggregateHandler` is that these objects do need to be provided for each command and event. The difference between the command handler and the event handler can be found in the following two things:

- Return values; a command handler returns a `Result` object. The event handler the new state of the aggregate.
- The context object

To prevent method signatures containing overly complex return objects, it had been decided to implement a context object. The return values for these handlers now only contains the single most crucial piece of information for both operations, while all other optional operations can be accessed through the context object. While the event handler can only access the current aggregates state through this context object, the command object can access the aggregate state, and in addition can stage events for future evaluation as well.

```csharp
public interface ICommandHandlerContext<out TAggregate>
	: IProvideAggregateInstance<TAggregate>
	where TAggregate : class, IAggregate
{
	public ReadOnlyCollection<IEvent> Events { get; }

	public void StageEvent(IEvent @event);
}

public interface IEventHandlerContext<out TAggregate>
	: IProvideAggregateInstance<TAggregate>
	where TAggregate : class, IAggregate
{

}
```

> Through the use of the context object we're able to provide a somewhat consistent developer experience across the numerous handler types we have. Later we will see such context object being used for services and sagas as well.

#### Services
The construction of a service is self-similar to the way a command is constructed. 

```csharp
public interface IServiceHandler
{
	public Task<Result> Handle<TService>(IServiceHandlerContext context, TService service)
		where TService : class, IService;
}
```

The main difference are the extended capabilities of a service, which becomes most clear when taking a look at the `IServiceHandlerContext` interface:

```csharp
public interface IServiceHandlerContext
{
	public ReadOnlyCollection<ICommand> Commands { get; }

	public Task<Result> EvaluateService<TService>(TService service)
		where TService : class, IService;

	public void StageCommand(ICommand command);
}
```

Since the idea is that services can be locally evaluated into a set of commands representing the side-effects of the service evaluation, the intent is to only record commands resulting from the evaluation of the service. This is also for the different way in which a service and a command can be dealt with from a service. In order to retrieve a list of commands a service must be evaluated. This is not the case for a command, which is the root cause for the difference in terminology found in the `IServiceHandlerContext`.

#### Sagas
The saga breaks with the patterns above in the sense that it does not contain a handler object. Since a saga solely responds to events that had been generated in the system and translates this to zero or more commands as continuation, the only thing which defines a saga is the translation between incoming event and outgoing commands.

```csharp
public interface ISaga<TAggregate, TEvent>
	where TAggregate : class, IAggregate
	where TEvent : IEvent
{
	public void Evaluate(ISagaContext<TAggregate> context, TEvent @event);
}
```

When it comes to this self-similarity we see the context object once again which in this case is tailored to the specific behaviour necessary from a saga.

```csharp
public interface ISagaContext<TAggregate>
	: IProvideAggregateInstance<TAggregate>
	where TAggregate : class, IAggregate
{
	public TAggregate? PreviousAggregateState { get; }

	public void StageService(IService service);
	public void StageCommand(ICommand command);
}
```

### Miscellaneous components
In addition to the handlers covered before there are a number of components with a special purpose in the domain model which will be provided under this section.

#### IAggregateProvider
The aggregate provider is a special component which is intended as the glue between the aggregate and the aggregate handler. The problem is that the aggregate handler by default is not coupled with a concrete aggregate instance. Providing this aggregate instance with every use of the aggregate handler would prove to be error prone. To work around this the aggregate provider is intended to abstract this responsibility. As such the interface of this object is similar to the interface of the aggregate handler, though without arguments for the aggregate instance.

```csharp
public interface IAggregateProvider
{
	public Task<Result<IEvent[]>> Evaluate(params ICommand[] commands);
	public Task Apply(params IEvent[] events);
}
```

