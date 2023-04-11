---
title: "Can I map one command to multiple events?"
slug: "mapping-one-command-to-multiple-events"
date: "2022-01-27"
summary: "Since it would be beneficial for commands to be able to issue multiple events reflecting its behaviour I modified an earlier approach to event sourced aggregates to allow this use-case."
references: 
  - '[[202201200000 event-sourced-aggregates]]'
  - '[[202201270000 coarse-commands-emitting-granular-events]]'
---

#software-development #dotnet

In two previous posts I already wrote about [the development of an event sourced aggregate](https://www.corstianboerman.com/blog/2022-01-20/event-sourced-aggregates), as well as [transparent command to event transformation](https://www.corstianboerman.com/blog/2022-01-27/more-transparent-command-to-event-transformation). Additionally I wrote a reflection about [the benefit of using a one to many mapping between commands and events](https://www.corstianboerman.com/blog/2022-01-27/coarse-commands-emitting-granular-events).

One of the key pieces of feedback I received was the assertion that it was not possible to return multiple events from the invocation of a single command. Since that is undoubtedly true I'll show how one or more events can be emitted using a collection of interfaces.

> I'd recommend you to check out [this post about designing event sourced aggregates](https://www.corstianboerman.com/blog/2022-01-20/event-sourced-aggregates) to read more about the starting point of this endeavour, if you haven't already.

Though it would be possible to wrap the `IEvent<T>` originally returned by the command in an `IEnumerable`, this approach would prove to be unusable in real world scenarios. The underlying reason is that type information is discarded, and therefore you'd have to try and cast the returned `IEvent` instance to its appropriate type to extract the relevant information.

Perhaps worse than that is the impossibility to guarantee a stable return type. Since the only constraint is that it must return a type of `IEvent`, it may simply return any event for said aggregate. Though not so much a problem for the way the domain itself behaves, this becomes rather problematic when being dependent on the domain to implement a certain behaviour. It'd require defensive coding techniques to be able to work with such interface while never nearly achieving the safety we could get when the return type is known.

## Updating interfaces
A more appropriate approach which can be suitable for production use is the following one. Based on the approach outlined in [this post which describes a more transparent command to event transformation process](https://www.corstianboerman.com/blog/2022-01-27/more-transparent-command-to-event-transformation), we're able to return multiple events from a single command while preserving type information.

Though the downside of this approach is a little bit of overhead on strongly typed interfaces determining what we can and cannot do, the benefits far outweigh the costs of doing so. Additionally this is an approached used within the .NET framework itself, therefore proving it will not hinder real world productivity.

To recap the transparent command we came up with previously:

```csharp
public interface ICommand<TState, TEvent>
	where TState : Aggregate<TState>
	where TEvent : IEvent<TState>
{
	public TEvent Validate(TState state);
}
```

This same approach can be extended to support multiple events being returned via a tuple. Whereas in the example below only two events are returned, multiple interfaces can be implemented to support an indefinite (?) number of events.

```csharp
public interface ICommand<TState, TEvent1, TEvent2>
	where TState : Aggregate<TState>
	where TEvent1 : IEvent<TState>
	where TEvent2 : IEvent<TState>
{
	public (TEvent1, TEvent2) Validate(TState state);
}
```

To ensure the aggregate is compatible with this altered command we'll need to write a bit of logic to allow the validation of the command against the domain.

```csharp
public abstract class Aggregate<T>
	where T : Aggregate<T>
{		
	public E Validate<E>
		(ICommand<T, E> @command)
			where E : IEvent<T>
		=> command.Validate((T)this);

	public (E1, E2) Validate<E1, E2>
		(ICommand<T, E1, E2> @command)
			where E1 : IEvent<T>
			where E2 : IEvent<T>
		=> command.Validate((T)this);

	public void Apply(IEvent<T> @event)
		=> @event.Apply((T)this);
	
	// Now we can support the application of an indefinite
	// number of events against the aggregate :)
	public void Apply(params IEvent<T>[] events)
		=> events.ToList().ForEach(Apply);
}
```

## Creating commands
The result of this is that we can now create commands which emit multiple events. In a minimal example a command could now look like this:

```csharp
public class UserUpdatesDetails
{
	public class UpdateDetails : ICommand<User, AddressUpdated, EmailUpdated>
	{
		public (AddressUpdated, EmailUpdated) Validate(User state)
		{
			return (new AddressUpdated(), new EmailUpdated());
		}
	}

	public class AddressUpdated : IEvent<User>
	{
		public void Apply(User state)
		{
			throw new NotImplementedException();
		}
	}

	public class EmailUpdated : IEvent<User>
	{
		public void Apply(User state)
		{
			throw new NotImplementedException();
		}
	}
}
```

Using said command against the user would then look like this:

```csharp
var user = new User();

var (addressUpdated, emailUpdated) = user.Validate(new UserUpdatesDetails.UpdateDetails());

user.Apply(addressUpdated, emailUpdated);
```

