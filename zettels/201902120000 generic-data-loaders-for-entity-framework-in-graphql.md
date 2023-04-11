
---
title: "How to create GraphQL data loaders for Entity Framework?"
slug: "generic-data-loaders-for-entity-framework-in-graphql"
date: "2019-02-12"
summary: ""
---

#software-development #dotnet #graphql #data-storage

> This article contains specific information potentially useful for users of both the GraphQL .NET and Entity Framework Core libraries. This article described the process to get by the result. If you just want inspiration for a generic data loader, feel free to skip to the end.

GraphQL is an amazing framework for creating (re)usable API's. Don't ask me why, but I ended up stuck with Entity Framework Core as my data access layer. When you are trying to develop a well thought out, usable and performant API you will get to use the data loader rather sooner than later.

Figuring out how to work with the data loader I absolutely refuse to write the same kind of code over and over again. I did not feel like writing a different data loader resolver for each and every entity type I need to retrieve in my API. If this is also the case for you, read on!

## The dataloader itself

> Feel free to skip this part if you know about the dataloader. This part is mainly for people just starting out with the dataloader, or completely unfamiliar with it.

Just to refresh on the data loader mechanism shipped with the GraphQL.net library. This mechanism is only there to help you retrieve data somewhat efficiently in batches. Imagine the following data structure (from one of my hobby projects):

```graphql
flight {
    departure {
        airfield {
            name
        }
        time
    }
    arrival {
        airfield {
            name
        }
        time
    }
}
```

The departure and arrival objects are of the same type, but represent a different piece of information. The way this information would be retrieved without dataloader is as follows:

1. All queried flights are retrieved from the database
2. For each flight result a database request for departure information is made
   * This departure type requests information about the airfield this flight departed from
3. For each flight another request for it's arrival information is made
   * For which a request for airfield information is made again.

For a single result in the result set (which is usually about 20 items long), about 5 requests need to be made. For a full page of information this results in 100 database roundtrips, which obviously is not performant.

> Assuming we have a 15ms latency to the database server from our dev machine, and each database call takes about 5ms, this would become a 2 second call. All but performant.

By using the dataloader to batch our requests based on the entities we are loading we can reduce this to three different calls (3% of the previous number of roundtrips):

1. A call for the flight entities
2. A request to retrieve flight information (departure and arrival)
3. A request for airfield information

We'll get to the technical implementation of a dataloader later in the aricle.

## What do we need?

There is something common about every data loader I have ever written. It's either one of the following two:

* Retrieve a set of records based on an equal or bigger set of property values
* Retrieve many records based on a smaller set of property values

Practical examples include these queries:

```csharp
q => ids.Contains(q.Id)
```

or

```csharp
q => ids.Contains(q.ParentId)
```

The differences are in the way we load the data. The first one will usually have a 1:1 mapping, while the 2nd example will usually involve a 1:many relationship. The common ground is that we always aim to retrieve a single entity type. There are only a few differences in each data loader method we usually have to write manually:

1. The type of data loader being initialized
2. The type for which we create the dataloader
3. The predicate provided

## Working with expression trees

LINQ queries are totally amazing! I love them, and I'd prefer to use these above weakly typed strings any time. In my GraphQL endpoints I only care about the data I want to request, and not so much the implementation details related to the dataloader. Therefore my goal: I want to provide a predicate, and a value for provided predicate, and retrieve all records where this evaluates to true.

> I found using LinqPad highly beneficial while working with expression trees. You can easily dump the result of a subexpression to see what you're working with.

The predicate shall be provided in the `Expression<Func<T, TKey>>` form. This could be used to provide a LINQ predicate like `q => q.Id`.

We face two problems:

1. In order to be able to query data we need to use the `Expression<Func<T, TKey>>` in a way that we get to a `Expression<Func<T, bool>>` in order to use it as argument in the `IQueryable<T>.Where(Expression<Func<T, bool>>)` method. It can be compared with writing a `T.Where(q => q.Id == 1)` query.
2. We do not only need to write a where query, but we need to compare values from a database to a list we created locally. (EF Core will translate this into a SQL `IN ('', '', ...)` statement.)

The combination of those two is interesting because we are going to be calling value methods through expression trees.

The code we would usually be writing will look a bit like this:

```csharp
dbSet.Where(q => list.Contains(q.Id));
```

Let's go through our variant built with expression trees.

### Calling the contains method

```csharp
var containsExpression = Expression.Call(
    Expression.Constant(ids),
    typeof(ICollection<TValue>)
        .GetMethod("Contains", new[] { typeof(TValue) }),
    predicate.Body
);
```

The `Expression.Call` method takes 3 arguments:

1. The object to call a method on (of the type `ICollection<TValue>`)
2. The method to call (on a type of `ICollection<TValue>`, with a single argument of `TValue`)
3. The arguments (in this case of `TValue`) to invoke the method with.

If we were to dump the contents of the predicate body in LinqPad we can see what it could look like:

![Expression body contents](/uploads/7_BA_362_A8_A2_4_FCA_421_B_BFE_8_348751_EFE_53_B_7_D_png_f0d3859d99.jpg)

The result we have now would look like the `list.Contains(q.Id)` section of a linq query. It will form the body of our LINQ final query.

### Building the full expression

What's left for us to do is to wrap the call in a where clause, and provide an actual value to run the expression with. This actual value will be provided by the `.Where` linq method.

```csharp
var expression = Expression.Lambda<Func<T, bool>>(
    containsExpression,
    predicate.Parameters
);
```

As `predicate.Parameters` seems pretty vague at first sight; it's nothing more than this data structure, which tells us variable `q` represents the type of `Flight` (it can be any type you want it to be if it's generic):

![predicate.Parameters](/uploads/7_B818_E3_F8_B_A8_C8_42_EB_9_BAE_4_A6_FCBFB_2334_7_D_png_0f8651f403.jpg)

## The full result

So to make a quick recap, these are the extension methods I am using to create a data-loader for a certain type.

```csharp
public static class GenericDataLoader
{
    public static Expression<Func<T, bool>> MatchOn<T, TValue>(
        this ICollection<TValue> items,
        Expression<Func<T, TValue>> predicate)
    {
        return Expression.Lambda<Func<T, bool>>(
            Expression.Call(
                Expression.Constant(items),
                typeof(ICollection<TValue>).GetMethod("Contains", new[] { typeof(TValue) }),
                predicate.Body
            ),
            predicate.Parameters
        );
    }

    /// <summary>
    /// Register a dataloader for T by the predicate provided.
    /// </summary>
    /// <typeparam name="T">The type to retrieve from the DbSet</typeparam>
    /// <typeparam name="TValue">The value to filter on</typeparam>
    /// <param name="dataLoader">A dataloader to use</param>
    /// <param name="dbSet">Entity Framework DbSet</param>
    /// <param name="predicate">The predicate to select a key to filter on</param>
    /// <param name="value">Value to filter items on</param>
    /// <returns>T as specified by the predicate and TValue</returns>
    public static async Task<T> EntityLoader<T, TValue>(
        this IDataLoaderContextAccessor dataLoader,
        DbSet<T> dbSet,
        Expression<Func<T, TValue>> predicate,
        TValue value)
        where T : class
    {
        if (value == null) return default;

        var loader = dataLoader.Context.GetOrAddBatchLoader<TValue, T>(
            $"{typeof(T).Name}-{predicate.ToString()}",
            async (items) =>
        {
            return await dbSet
                .AsNoTracking()
                .Where(items
                    .ToList()
                    .MatchOn(predicate))
                .ToDictionaryAsync(predicate.Compile());
        });

        var task = loader.LoadAsync(value);
        return await task;
    }

    /// <summary>
    /// Register a dataloader for an IEnumerable<T> by the predicate provided.
    /// </summary>
    /// <typeparam name="T">The type to retrieve from the DbSet</typeparam>
    /// <typeparam name="TValue">The value to filter on</typeparam>
    /// <param name="dataLoader">A dataloader to use</param>
    /// <param name="dbSet">Entity Framework DbSet</param>
    /// <param name="predicate">The predicate to select a key to filter on</param>
    /// <param name="value">Value to filter items on</param>
    /// <returns>IEnumerable<T> as specified by the predicate and TValue</returns>
    public static async Task<IEnumerable<T>> EntityCollectionLoader<T, TValue>(
        this IDataLoaderContextAccessor dataLoader,
        DbSet<T> dbSet,
        Expression<Func<T, TValue>> predicate,
        TValue value)
        where T : class
    {
        if (value == null) return default;

        var loader = dataLoader.Context.GetOrAddCollectionBatchLoader<TValue, T>(
            $"{typeof(T).Name}-{predicate.ToString()}",
            async (items) =>
        {
            var compiledPredicate = predicate.Compile();

            return dbSet
                .AsNoTracking()
                .Where(items
                    .ToList()
                    .MatchOn(predicate))
                .ToLookup(compiledPredicate);
        });

        var task = loader.LoadAsync(value);
        return await task;
    }
}
```

To summarize above code:

* First there's the generic part which creates our expression tree (the `MatchOn` extension method)
* Then there's an extension method which enables you to retrieve a single item via the dataloader (the `EntityLoader`)
* Last but not least there's a method which enables you to retrieve a list of items based on a single value (the `EntityCollectionLoader`)

## What's next?

As I'm progressing with the GraphQL dotnet library I'm starting to understand more and more about the design principles used while building this framework. This enables me to abstract more logic away than ever before. Right now I'm working on some additional general purpose data-loaders and I'm planning to release them as NuGet package somewhere soon.

I'll let you know when this library is available.
