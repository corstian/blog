---
title: "More transparent command to event transformation"
slug: "more-transparent-command-to-event-transformation"
date: "2022-01-27"
summary: "An improvement upon the previously proposed event sourced aggregate where command provides type information about its returned domain events."
references: 
  - '[[202201200000 event-sourced-aggregates]]'
---

#software-development #dotnet

In [a previous post](https://www.corstianboerman.com/blog/2022-01-20/event-sourced-aggregates) I covered some ideas about the implementation of event sourced aggregates. A pressing issue that came to my attention is the way that the structure previously covered does not preserve type information on the commands, therefore making life of those using the domain unnecessarily difficult.

To recap what had previously been implemented:

```csharp
public interface ICommand<TState>
    where TState : Aggregate<TState>
{
    public IEvent<TState> Validate(TState state);
} 
```

```csharp
public interface IEvent<TState>
    where TState : Aggregate<TState>
{
    public void Apply(TState state);
}
```

Though these interfaces work well in a situation where the sole role of the event is to be persisted and applied to the internal state, it becomes rather difficult when dealing with user interaction. A hypothetical example about how the domain would currently be consumed.

```csharp
IEvent<T> @event = User.Validate(new UserCreation.CreateUser {
	Email = "john.doe@example.com"
});
```

The inherent difficulty here is knowing the type of the return value, which is known to the command, but not necessarily to those implementing the command. Therefore we'll need a more transparent approach to handle the command to event transformation.

## Adding type information
Type information can be added in a rather simple way on the command interface itself. Since the command is the type responsible for the translation into events, it makes sense to have it expose information about the return types.

```csharp
public interface ICommand<TState, TEvent>
	where TState : Aggregate<TState>
	where TEvent : IEvent<TState>
{
	public TEvent Validate(TState state);
}
```

Though the downside of this approach is an additional piece of boilerplate code, the benefits by far outweigh these considerations.

In order to support the command to event transformation of this newly created command we'll also need to implement these on the aggregate itself.

```csharp
public abstract class Aggregate<T>
	where T : Aggregate<T>
{
	public IEvent<T> Validate(ICommand<T> @command)
		=> @command.Validate((T)this);
	
	public E Validate<E>(ICommand<T, E> @command) 
		where E : IEvent<T>
		=> command.Validate((T)this);

	public void Apply(IEvent<T> @event) => @event.Apply((T)this);
}
```

After modifying the command to reflect this new interface including type information the way we're consuming the domain now looks like this from the perspective of an unit test:

```csharp
[Fact]
public void UserCreationTest()
{
	var user = new User();

	var createUserCommand = new UserCreation.CreateUser
	{
		Name = "John Doe",
	};

	// Properly recognized to be of type `UserCreated` rather than `IEvent<User>`
	var userCreatedEvent = user.Validate(createUserCommand);

	user.Apply(userCreatedEvent);

	Assert.Equal("John Doe", user.Name);
}
```

Therefore the way one could consume the domain now is as follows:

```csharp
UserCreation.UserCreated @event = User.Validate(new UserCreation.CreateUser {
	Email = "john.doe@example.com"
});
```

By providing the explicit domain event resulting from the command it becomes much more easier to implement public APIs on top of the domain.

## Bonus: Breaking Unit Tests
Changing the interface of the `CreateUser` command from that of `ICommand<TState>` to `ICommand<TState, TEvent>` only breaks half of the unit tests previously implemented, just like the following one:

```csharp
[Fact]
public void NameCannotBeNullOrWhiteSpace()
{
	ICommand<User> command = new CreateUser();

	Assert.Throws<Exception>(() => @command.Validate(new User()));
}
```

The reason for this is the explicit cast to `ICommand<User>`, which is required to show the explicitly implemented `Validate` method. Fixing the test however is super easy; we'll just have to let it reflect the currently used command interface.
