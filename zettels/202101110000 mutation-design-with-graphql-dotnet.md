---
title: "Mutation design with graphql-dotnet"
slug: "mutation-design-with-graphql-dotnet"
date: "2021-01-11"
summary: ""
references: 
  - '[[201903080000 implementing-pagination-with-graphql-net-and-relay]]'
---

#software-development #graphql


GraphQL is fairly flexible when it comes to API design. Sure, it's not as flexible as OData, though that may actually be a blessing. Maybe it's the restrictions in term of API structure which give GraphQL much of its power. In this post I'll cover another subtle method which will improve usability and therefore the quality of your API;


But first more about API design methodology; there seem to be two aproaches one can follow, which are dependent on the complexity of the project, as well as the intended features of an API.

1. Model based CRUD operations
2. CQRS style action oriented API

Though this technique will be applicable to both of these approaches, it will shine when you follow the action based approach, where you design your API to represent actions executed by users, instead of modifications to the underlying data model. Nevertheless, if you're planning on starting with a CRUD approach, but would like to move on to an action oriented approach later on, it might be valuable to prepare your API for that.

## The problem
Imagine the following mutation;

```
mutation {
  profile {
    create(name: "John Doe")
  }
}
```

Supposedly a model will be supplied instead of a single name, and with a CRUD based API, that's probably all you'll need to do. However, when you have a domain model which represents different actions, it may be cumbersome practice to have to write API endpoints which combine one or more of these actions in a single call. As such, one may need to execute two mutations in order to create a profile and to associate an additional piece of information with the profile that had just been created.

Instead I'd prefer to be able to chain these actions. A great design tidbit about GraphQL API's is that while all query nodes will be resolved asynchronously, all mutation nodes will be performed synchronously. The practical implication being that we're able to throw a bunch of actions in a single query, and have them resolve in the same order as we provided. But there's a problem.

What would happen if I were to create an API which has the following endpoints (as modelled using graphql-dotnet);

```csharp
Field<AircraftType>()
    .Name("create")
    .Argument<NonNullGraphType<AircraftOptionsInputType>>("input")
    .ResolveAsync(async context => { /*...*/ });

Field<AircraftType>()
    .Name("addIdentifier")
    .Argument<NonNullGraphType<StringGraphType>>("cursor")
    .Argument<NonNullGraphType<StringGraphType>>("identifier")
    .ResolveAsync(async context => { /*...*/ });
```

Ultimately I want to be able to combine these two operations in a single mutation, but there's a problem as the cursor is generated upon creation, and therefore I would be unable to invoke the `addIdentifier` operation until I get the result of the `create` operation back. This technique would require two roundtrips to the server, and even worse for me, it'd significantly increase complexity in any frontend I'd be consuming these in.

There are two solutions I can employ;

1. Solve this on the client side and pre-generate an identifier passed onto the `create` operation.
2. Wrap these operations in an object which can share information about the identifier with other operations.

The first approach I suggested would be a terrible idea. It'd require the client to know about the way I generate cursors, which is exactly the thing I've been trying to prevent for using cursors. The second would require a little bit more complexity on the server side, but is actually implementable within five minutes.

## The solution
> There is one important design decision within my data store which enables this approach, and that is that I opted for the use of GUID's as identifiers. Back in the day I had done this with the idea in mind that it'd make scaling significantly easier, but a side effect is that it'll allow identifier generation outside of the data store.

Assume the following class as the root mutation for my API;

```csharp
public class GraphMutation : ObjectGraphType<object>
{
    public GraphMutation()
    {
        Field<AircraftMutations>()
            .Name("aircraft")
            .Argument<StringGraphType>("cursor")
            .Resolve(context => new ObjectWithCursor(context.GetArgument<string>("cursor")));
    }
}
```

Important enough, instead of providing an empty object to instantiate the `AircraftMutations` object, I'm providing an `ObjectWithCursor` object. This works just exactly the same way as you'd otherwise design query objects using graphql-dotnet.

The `ObjectWithCursor` object is defined as follows;

```csharp
public class ObjectWithCursor
{
    public ObjectWithCursor(string cursor)
    {
        if (string.IsNullOrWhiteSpace(cursor))
        {
            Cursor = Guid.NewGuid();
            WasCursorProvided = false;
        } else
        {
            Cursor = cursor.FromCursor();
            WasCursorProvided = true;
            
            /* 
             * For me a cursor represents a base64 encoded guid.
             *
             * The FromCursor extension method:
             * public static Guid FromCursor(this string base64) => new Guid(Convert.FromBase64String(base64));
             */
        }
    }

    public Guid Cursor { get; }
    public bool WasCursorProvided { get; }
}
```

As mentioned before, the benefit of having GUID's as identifier is that you can generate new identifiers outside of the database itself. As such I'm creating one in case none is provided, which I will use as primary key when a new object is generated. The `AircraftMutations` object may look like that shown beneath;

```csharp
public class AircraftMutations : ObjectGraphType<ObjectWithCursor>
{
    public AircraftMutations()
    {
        Field<AircraftType>()
            .Name("create")
            .Argument<NonNullGraphType<AircraftOptionsInputType>>("input")
            .ResolveAsync(async context =>
            {
                if (context.Source.WasCursorProvided)
                    throw new ArgumentException("Object creation with predefined cursor is not supported");

                /* Continue creating a new object */
                throw new NotImplementedException();
            });

        Field<AircraftType>()
            .Name("addIdentifier")
            .Argument<NonNullGraphType<StringGraphType>>("identifier")
            .ResolveAsync(async context =>
            {
                var cursor = context.Source.Cursor;
                var value = context.GetArgument<string>("identifier");

                /* Continue modifying the aircraft object */
                throw new NotImplementedException();
            });
    }
}
```

In this case the identifier is extracted from the parent object, and can be used to combine multiple operations in a single mutations. Please be aware that this approach requires additional validation within fields to check whether the required arguments are available.

The queries we'll be able to write now look as follows;

```
mutation {
  aircraft {
    create(input: {
      registration: "PH-ABC",
      callsign: "ABC"
    })
    addIdentifier(identifier: "0xFFFFFF")
  }
}
```

## Recap
The approach as described in this post will enable the development of API's which are more flexible than otherwise possible, as one can combine as many combinations as the scope allows, without being limited by the cognitive complexity of handling multiple mutations from any front-end. As a side effect, the server side API can be kept much simpler, adhering to the DRY principles, while offering a great deal of flexibility in how the API can be used.
