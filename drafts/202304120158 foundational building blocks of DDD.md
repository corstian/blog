---
title: "Foundational building blocks of DDD"
slug: "foundational-building-blocks-of-ddd"
date: "2023-04-13"
summary: ""
references: 
alias:
  - foundational building blocks of DDD
---

Domain Driven Design, DDD for short, is a methodology focused on discovering, clarifying and technically expressing (business) processes. As part of this, DDD techniques include a vocabulary to express these processes in a coherent manner helping in the translation between the conceptual challenges and the technical representation of these. All taken together these components represent your domain.

## Main aspects
The aspects discussed below comprise most of the basic vocabulary to technically describe a software domain. Note that there is much more depth to it than the mere descriptions given here. In addition to these technical aspects there are also conceptual considerations to these components which are mostly ignored in this document.

### Value
A value is an immutable piece of data which can be shared by anyone needing to reference the same information contained in one. An address is a good example of information which can be contained within a value.

```csharp
public record AddressValue(
	string Address,
	string Zip,
	string City,
	string Country
);
```

Values can be shared among aggregates, and aggregates are able to store these values if they deem this necessary. This in stark contrast to entities, which should not be shared among aggregates.

### Entity
An entity is a mutable data object. Due to this mutability an entity can also have methods to alter the data contained within, but should therefore also only be used within an aggregate.

```csharp
public class ProfileEntity {
	public string Name { get; init; }
	public string Email { get; init; }

	public void ChangeName(string name) => Name = name;
	public void SetEmail(string email) => Email = email;
}
```

### Aggregate
An aggregate is perhaps the most important aspect from domain driven design. It's meaning can be thought of as the combination of two separate aspects:

1. Providing **a conceptual consistency boundary around a group of entities**. Entities within this boundary can thus only be mutated from within this boundary, and changes made within this boundary should not have side-effects outside this boundary.
2. Being the sole entity within the consistency boundary of the aggregate allowed to be accessed from the outside. This is also known as the "Aggregate Root", or "Aggregate" for short as well.

From the outside interactions to an aggregate should only be addressed to the aggregate root. Further entities contained by the entity should be hidden from the outside view. This ensures one is more free to alter the internals of the aggregate without introducing breaking changes.

```csharp
public class ProfileAggregate {
	private CreditCardInfoEntity CCInfo { get; init; } = new();

	public void SetCreditCardNumber(string number) {
		CCInfo.CCNumber = number;
	}

	public string LastFour => CCInfo.CCNumber.Substring(
		Math.Max(0, CCInfo.CCNumber - 4));

	private class CreditCardInfoEntity {
		// ... be careful with sensitive data
		internal string CCNumber { get; init; } = "";
	}
}
```

In the code snippet above we see that we are able to set the credit card number on the profile, though are only able to retrieve the last four characters for verification purposes. This is what an aggregate boundary is about; only expose what you need to.

### Service
The service is a component which, in contrast to an aggregate, may trigger side-effects beyond its own boundaries, or perhaps even beyond the boundary of the domain. When an operation should be coordinated across multiple aggregates a service is the entry point of choice.

Since services may also incur changes across their boundaries they can also trigger calls to external services such as databases and APIs. Sometimes an explicit distinction is made between pure domain services and application services, where the latter are placed outside of the core domain, but act as the glue between the domain and the external service.

Regardless of the way services are structured, it is important to prevent the core domain from being dependent on an external component. A common way this is decoupled is by defining an interface within the core domain describing the behavior of the external service, developing the domain service against the interface, and later supplying a concrete instance through dependency injection.

### Repository
A repository is a concrete example of the external dependency which is decoupled from the core domain. Repositories are mostly used to provide access to data, but the pattern can be repeated for other external concerns.

```csharp
public interface IRepository<T> {
	public T Get(Guid id);
	public void Set(T t);
}

// Then implement the concrete behaviour outside of the core domain

public class ThisIsAService {
	private readonly IRepository<ProfileAggregate> _repo;
	
	public ThisIsAService(IRepository<ProfileAggregate> repo) {
		_repo = repo;
	}

	public void DoSomething() {
		var profileA = _repo.Get(Guid.NewGuid());
		var profileB = _repo.Get(Guid.NewGuid());
	
		// Now do something with both profiles
	}
}
```

### Bounded Context
The bounded context provides a consistency boundary around a portion of the domain. Conceptually this allows one to assign a different meaning to the same terminology, dependent on context. This is especially useful for different teams working a different part of the business and therefore having a different understanding of the business. This helps these teams to be internally consistent, and only to resolve these differences in meaning on the boundaries of the bounded contexts, which are the integration points between these subsystems.

## Extended aspects
In addition to the primary vocabulary described above there are also some components which might be useful depending on system architecture and desirable behavioural aspects.

### Commands / Events
Though generally associated with event sourcing techniques, the use of commands and events is a powerful abstraction over something which is normally expressed implicitly within aggregates.

Where the command is the request to do a certain operation, the event is the confirmation of said operation. When the command is the validation of invariants, the event represents the state change.

Such abstraction allows one to properly abstract the aggregate as well allowing one to implement generalized behaviour across all domain components.

See [[202201200000 event-sourced-aggregates]] for an example of this approach.

### Sagas
A saga is a component known from event driven systems. In these, a saga is a continuation on an event, and translates an incoming event into a new follow-up operation. In the context of domain driven development, the saga would be the temporal continuation of an operation. In this sense it is an useful component for the integration between different bounded contexts.

The saga must not be confused with the process manager. While both are intended to handle events, the saga does not hold any state and must deal with whatever is provided to it through events. Process managers do not have this limitation, but then these can also be built by creating an aggregate with such specific purpose.