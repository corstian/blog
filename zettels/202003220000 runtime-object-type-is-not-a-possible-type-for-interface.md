---
title: "GraphQL.NET: Runtime object type is not a possible type for interface"
slug: "runtime-object-type-is-not-a-possible-type-for-interface"
date: "2020-03-22"
summary: ""
references: 
  - [[201903110000 how-to-automatically-load-graph-types]]
  - [[201903080000 implementing-pagination-with-graphql-net-and-relay]]
---

#software-development #dotnet #graphql

This is primarilly a note to myself, for the possible future where I start searching for this error again, in the changes of saving me another day of debugging.

## Problem

This error occurs when you have an `InterfaceGraphType`, an appropriate implementation of `ObjectGraphType` which refers back to the interface, but the following error being returned when resolving into the implementing `ObjectGraphType`.


In the example below I am trying to resolve the `PaymentMethodInterface` into the concrete type of `CardPaymentMethodType`.

```json
{
  "data": {
    "profile": {
      "paymentMethod": null
    }
  },
  "errors": [
    {
      "message": "GraphQL.ExecutionError: Runtime Object type \"CardPaymentMethodType\" is not a possible type for \"PaymentMethodInterface\".\r\n   at GraphQL.Execution.ExecutionStrategy.ValidateNodeResult(ExecutionContext context, ExecutionNode node)\r\n   at GraphQL.Execution.ExecutionStrategy.ExecuteNodeAsync(ExecutionContext context, ExecutionNode node)",
      "locations": [
        {
          "line": 3,
          "column": 5
        }
      ],
      "path": [
        "profile",
        "paymentMethod"
      ]
    }
  ]
}
```

## Solution

The solution is actually described [in the GraphQL documentation over here](https://graphql-dotnet.github.io/docs/getting-started/interfaces#registertype). As it states:

> "When the Schema is built, it looks at the "root" types (Query, Mutation, Subscription) and gathers all of the GraphTypes they expose. Often when you are working with an interface type the concrete types are not exposed on the root types (or any of their children). Since those concrete types are never exposed in the type graph the Schema doesn't know they exist."

The idea this problem would be automatically tackled by either the `.AddGraphQL()` or `.AddGraphTypes()` extension methods on the `IServiceCollection`, like I had, is wrong.

In order to resolve this issue, the only thing you have to do is call the `ResolveType<T>()` method within the constructor of your schema class. In my specific case this would be `RegisterType<CardPaymentMethodType>();`.
