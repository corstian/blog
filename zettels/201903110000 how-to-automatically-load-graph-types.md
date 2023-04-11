---
title: "How to automatically load graph types in the DI container."
slug: "how-to-automatically-load-graph-types"
date: "2019-03-11"
summary: ""
references: 
  - '[[202003220000 runtime-object-type-is-not-a-possible-type-for-interface]]'
  - '[[201903080000 implementing-pagination-with-graphql-net-and-relay]]'
  - '[[201902120000 generic-data-loaders-for-entity-framework-in-graphql]]'
---

#software-development #dotnet #graphql

[GraphQL.NET](https://github.com/graphql-dotnet/graphql-dotnet) relies a lot on DI containers to get instances for certain graph types. At first it might be a bit daunting to come across an error along the likes of this:

```
GraphQL.ExecutionError: No service for type '...' has been registered.
```

It is GraphQL's way of telling the world that it could not find an instance of a certain type in the DI container.

## Solving it the easy way

The usual way to solve this problem is to register a new instance with the DI container which usually goes like this:

```csharp
services.AddSingleton<SomeQuery>();
```

It works. But for large GraphQL api's it doesn't scale well, and it's annoying for the developers to be remembered to register types with the DI container everytime a new type has been added.

## Solving it for once and for all

Due to habit I tend to cluster the same types of classes together in the same folder/namespace.

My folder structure looks a bit like this:

```
Graph
|- Types
   |- Enum
   |- Input
   |- Interface
   |- Object
```

The nice thing about this is that all different types of `ObjectGraphType` derivatives are stored in the `Graph.Types` namespace, while also having a distinction between the several subtypes.

Now in order to solve the registration problem for once and for all you can add the following code block to the place where you usually register your graph types.

```csharp
Assembly
    .GetExecutingAssembly()
    .GetTypes()
    .Where(q => q.IsClass
        && (q.Namespace?.StartsWith("Graph.Types") ?? false))
    .ToList()
    .ForEach(type => services.AddSingleton(type));
```

The code shown above invokes some reflection magic to retrieve all classes in the executing assembly (which in my case is also the assembly in which I have defined the graph types), filter them for the namespace, which is supplied as string, and add them to the DI container.

The only thing you need to do is to replace the namespace, and verify whether you pull your metadata from the correct assembly.

> *Use of reflection is being scrutinized because it is 'slow' and there are usually better alternatives. In this case it will lead to more maintainable code, and it is only ran during startup either way ¯\\_(ツ)_/¯*
