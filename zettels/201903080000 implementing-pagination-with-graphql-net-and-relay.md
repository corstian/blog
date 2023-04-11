---
title: "Implementing pagination with GraphQL.NET and Relay"
slug: "implementing-pagination-with-graphql-net-and-relay"
date: "2019-03-08"
summary: "I describe the implementation of a cursor based GraphQL API following the Relay specification. Everything is covered, the specification, query composition, and even a real-world sample."
references: 
  - [[202101110000 mutation-design-with-graphql-dotnet]]
  - [[202003220000 runtime-object-type-is-not-a-possible-type-for-interface]]
  - [[201912160000 graphql-dotnet-authorization]]
  - [[201910240000 graphql-api-design-cursors]]
  - [[201903110000 how-to-automatically-load-graph-types]]
  - [[201809100000 asp-net-core-2-1-and-graphql-adding-jwt-bearer-validation-to-subscriptions]]
  - [[201902120000 generic-data-loaders-for-entity-framework-in-graphql]]
  - [[201812260000 cursor-based-pagination-with-c-and-sql-server]]
  - [[201903060000 cursor-based-pagination-with-sql-server]]
---

#software-development #dotnet #sql #graphql #data-storage

> **Trigger warning:**  
> *In this article I am bashing quite a bit on traditional REST api's. Not that you can't have a great architecture with REST api's, but from my experience these types of API's are badly implemented more often than not. Same goes for the tech described in this article. If used incorrectly you'll end up voluntary amputating one of your limbs.*
>
> *As a friend noted: "Talk to REST and get a certain object, or ask GraphQL and retrieve some information".*

---

## <a href="#ic_contents" id="ic_contents">0.</a> Contents

1. [Introduction](#ic_introduction)  
2. [Background](#ic_background)  
  2.1 [Cursor Based Pagination](#cursor-pagination)  
  2.2 [Connection Specification](#connection-specification)  
  2.3 [Specification, but simplified](#simplified-specification)  
  2.4 [The `Connection`'s Structure](#connection-structure)  
  2.5 [Generating Cursors](#generating-cursors)
3. [Overview](#ic_overview)
4. [Architecture](#architecture)  
5. [The GraphQL Connection Endpoint](#graphql-connection-endpoint)  
6. [Data Retrieval](#data-retrieval)  
  6.1 [[Contextual] Model and Graph Types](#data-models)  
  6.2 [Resolving Arguments](#ic_arguments)  
  6.3 [Data Filters](#data-filters)  
  6.4 [Slicing for Pagination](#slicing)  
  6.5 [Creating the Connection object](#connection-object)
7. [Overview of a real-world implementation](#realworldsample)  
  7.1 [The Connection](#realworldconnection)  
  7.2 [Data access function](#realworlddataaccess)  
  7.3 [Defining order on a connection](#order)
8. [Other Resources](#ic_other-resources)

## <a href="#ic_introduction" id="ic_introduction">1.</a> Introduction

Pagination is always a topic that's a bit tricky to tackle. The goal is to give as much freedom as possible for (client side) queries, while staying in control of the data that goes out.

[GraphQL is no difference](https://github.com/graphql-dotnet/relay/issues/25), but thankfully the [Relay](https://facebook.github.io/relay/) extension has been developed on top of GraphQL. Relay consists of a set of conventions which solidifies the GraphQL specification a bit more so that client side tools can quickly scaffold application logic for said imaginative api.

For the assumptions Relay makes about a GraphQL server, see [this document](https://relay.dev/docs/guides/graphql-server-specification).

## <a href="#ic_background" id="ic_background">2.</a> Background

While [quite poorly documented in the GraphQL.NET docs](https://graphql-dotnet.github.io/docs/getting-started/relay) (at the time of writing), functionality for Relay compatible endpoints has been implemented. [Some of this logic](https://github.com/graphql-dotnet/graphql-dotnet/tree/master/src/GraphQL/Types/Relay) resides in the main graphql-dotnet package, while other tools and helpers reside in [the relay package](https://github.com/graphql-dotnet/relay).

Because Relay is only a specification of assumptions about interfaces and available methods, it is fairly easy to implement yourself. Consider most of the helper methods provided by the [graphql-dotnet/relay](https://github.com/graphql-dotnet/relay) package as suggestions, and think about writing specialized variants which suit your use-cases. More on that later.

> *If you are already familiar with the design philosophy that goes with GraphQL, and indirectly with the graphql-dotnet package, feel free to skip to the [overview](#ic_overview).*

### <a href="#cursor-pagination" id="cursor-pagination">2.1</a> Cursor Based Pagination

The biggest part of the Relay specification consists of details on the `Connection` types. A connection is a type which which describes a list of objects, and arguments to filter and slice this list. The arguments available in a connection by default:

* **`after`**: Accepts a cursor and describes that entities after this one should be retrieved.
* **`first`**: The number of items to retrieve after given cursor.
* **`before`**: Accepts a cursor and describes that entities before this one should be retrieved.
* **`last`**: The number of items to retrieve starting from the back (or before a given cursor).

These 4 arguments make up for a powerful concept which is called cursor based pagination. With this model every object has an unique identifier (you most certainly already have one already anyway), which you can use in order to create slices of data.

These slices of data are relative to one another. For the first call to the api you could provide the `first` argument to a connection. For all subsequent calls you would use the cursor of the last element in this list to retrieve a new slice of data.

This type of pagination, although it takes a bit more work implementing than traditional offset based pagination, is a delight to work with. On the client side it allows you to implement both traditional page based pagination methods, as well as more progressive and user-friendly paradigms. Another improvement over offset based pagination is that this method works fine with realtime data. A lot of complexities that are inherent to combining realtime data and offset based pagination just simply disappear.

> If you look on Google you'll find plenty of reasons why we should stop doing offset based pagination. Honestly I do not care about the technical implications at all, but the thing is that it's just not user-friendly. [The beginning of this article](https://blog.jooq.org/2016/08/10/why-most-programmers-get-pagination-wrong/) digs into that a bit deeper.

### <a href="#connection-specification" id="connection-specification">2.2</a> Connection Specification
For a detailed specification about connections and cursors, see this document: https://facebook.github.io/relay/graphql/connections.htm. Item [4.3 of the Relay specification](https://facebook.github.io/relay/graphql/connections.htm#sec-Pagination-algorithm) covers expected behaviour of the pagination algorithm.

### <a href="#simplified-specification" id="simplified-specification">2.3</a> Specification, but simplified

The specification describes the way data should be retrieved. The following steps could be followed:

1. Apply your data filtering operations
2. If `after` is specified:
    - Take all data starting from this cursor
3. If `before` is specified:
    - Take all data before this cursor
4. If `first` is specified:
    - Take the specified amount of entities from the beginning.
5. If `last` is specified:
    - Take the specified amount of entities from the end.
6. Return data

The specification defines the cases where all parameters (`after`, `before`, `first` and `last`) are specified. There are few use-cases where using all these parameters at the same time will make any sense at all. It does provide some consistency and predictability to the use of cursors, though.

While it is possible to run all filter operations on the .NET runtime, I would totally NOT recommend doing so. It would have the implication that all data has to be loaded locally, which does seem to result in the infamous `OutOfMemoryException` in my case. If it works for you now but your data starts scaling later, it would be incredibly difficult to keep it all running smoothly after a while. I recommend to invest some time researching the different options for cursor based pagination within your data provider. E.g. MsSql, MySql, ElasticSearch or whatever your tool of choice is. If you still decide you want to shoot your own foot off and do pagination in-memory, go ahead. There are some methods available in [the graphql-dotnet/relay project](https://github.com/graphql-dotnet/relay/blob/fe25a75b525f2eb0fa83e5b7dd9a23c7f1a93de4/src/GraphQL.Relay/Types/ConnectionUtils.cs) for which I have created the following extension method:

```csharp
public static Connection<TSource> ToConnection<TSource, TParent>(
    this IEnumerable<TSource> items,
    ResolveConnectionContext<TParent> context,
    int sliceStartIndex,
    int totalCount,
    bool strictCheck = true)
{
    return ToConnection(items, context, sliceStartIndex, totalCount, strictCheck);
}
```

But again, this is definitely the unfun way to do it! If this is all you've been looking for you can close this article now. This article will further elaborate on offloading these filter and slice operations to an external data-store.

### <a href="#connection-structure" id="connection-structure">2.4</a> The `Connection`'s Structure

The connection is not only these four arguments you can slice your data with. Another part of the connection types are some subtypes to help query through the data. The main structure looks as follows:

```
friends(first: 10, after: "opaqueCursor") {
  edges {
    cursor
    node {
      id
      name
    }
  }
  pageInfo {
    hasNextPage
  }
}
```
*As taken from [this example](https://facebook.github.io/relay/graphql/connections.htm)*.


Connection endpoints created with the graphql-dotnet project have the following default structure ([see the `ConnectionType` class](https://github.com/graphql-dotnet/graphql-dotnet/blob/master/src/GraphQL/Types/Relay/ConnectionType.cs)):

- [`edges`](https://facebook.github.io/relay/graphql/connections.htm#sec-Edges): Is a list containing so called [`EdgeType`](https://facebook.github.io/relay/graphql/connections.htm#sec-Edge-Types)'s. 
  - `cursor`: The opaque cursor used to uniquely identify a resource.
  - `node`: The concrete object you're trying to return from your connections. In case of the example above it would probably be represented by the `FriendType` class.
- [`pageInfo`](https://facebook.github.io/relay/graphql/connections.htm#sec-undefined.PageInfo): This type contains metadata about the available data.
  - `hasPreviousPage` *(required)*: 
  - `hasNextPage` *(required)*:
  - `startCursor`: [The graphql-dotnet project has added this field.](https://github.com/graphql-dotnet/graphql-dotnet/blob/master/src/GraphQL/Types/Relay/PageInfoType.cs) It contains the value of `edges[0].cursor`.
  - `endCursor`: See above
- `totalCount`: [Also something added in the graphql-dotnet project.](https://github.com/graphql-dotnet/graphql-dotnet/blob/master/src/GraphQL/Types/Relay/ConnectionType.cs) Personally I use this field to show the total number of results, considering the filters which have been applied.

### <a href="#generating-cursors" id="generating-cursors">2.5</a> Generating Cursors

It's important to note that the cursors used in a GraphQL connection are defined as a `string` type. This means that you can virtually put anything in there. For consistencies sake, and in order to keep the back-end and the front-end totally separated, it's recommend to encode your 'primary keys' in a base64 representation. This makes the front-end less reliant on your primary key type, and allows you to switch or combine many different data sources in a consistent way.

After all designing api's involves a lot of human interaction design.

As most of my infrastructure is build around `Guid`'s as unique identifier I have two simple extension methods which help me a lot when dealing with GraphQL endpoints:

```csharp
public static string ToCursor(this Guid guid)
    => Convert.ToBase64String(guid.ToByteArray());

public static Guid FromCursor(this string base64Guid)
    => new Guid(Convert.FromBase64String(base64Guid));
```

## <a href="#ic_overview" id="ic_overview">3.</a> Overview
It's quite difficult to think about pagination in combination with GraphQL. Like always we have to choose a balance between performance, quality and cost. There is a combination of issues which makes the problems look fuzzy:

- Ideally we want to be able to dynamically filter our results
- The (paginated) result set should be sliced over the data which already has been filtered

By separating these issues you can write more powerful, and reusable code which easily plugs into GraphQL. The most interesting side effect is that the resulting code will be incredibly easy to maintain (again, when compared to the traditional REST API paradigms).

The structure we build will contain:

- A data access method (usually per-type)
- An extension method to do some pagination
- The usual GraphQL wiring

In this guide we will walk through implementing connections in a GraphQL api using the [graphql-dotnet](https://github.com/graphql-dotnet/graphql-dotnet) project.

We assume you have the following NuGet packages installed (version numbers at the time of writing):

- [graphql-dotnet - 2.4.0](https://www.nuget.org/packages/GraphQL/2.4.0)
- [graphql-relay - 0.5.0](https://www.nuget.org/packages/GraphQL.Relay/0.5.0)

Besides that we assume you have your graph types and models set up properly.

> The convention I use is `ModelName` and `ModelNameType` to make the distinction between models and their respective graph types. As I have the habit of reusing my (Entity Framework) data models throughout my graph types, I usually ignore complex properties on the object. Resolving these other types is a task for which the data loader is perfectly suited.

## <a href="#architecture" id="architecture">4.</a> Architecture
The software architecture is one of the most important aspects in this article. The way you retrieve your data can either make or break your api. Throughout the implementation we will follow these core guidelines:

- **Don't Repeat Yourself (DRY)**: This one is a cliche, but really important. Everyone talks about this one, but in the REST world it was barely enforced, nor taken seriously at all. Look at the scaffolding logic Microsoft shipped with ASP.NET and Entity Framework back in the day. It did promote reusable software in a copy/paste kind of way. Not in a invoke-this-method-everywhere kind of way.
- **Single Point of Responsibility**: [Also known as the Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html). The code we write does one thing, and one thing good. We can divide between two main topics here: *data filtration* and *data slicing*.
- **Keep It Simple, Stupid (KISS)**: There's no reason to make stuff more complex than it needs to be. Interestingly enough it's quite easy to make stuff a lot more simple if you tackle a problem at macro level.

By applying these core rules we end up with a few components from technical perspective:

- The GraphQL API
- Data retrieval logic
- Data slicing behaviour

## <a href="#graphql-connection-endpoint" id="graphql-connection-endpoint">5.</a> The GraphQL Connection Endpoint
The GraphQL connection endpoints can be compared with traditional `Field` based ones. They show that the api offers access to a certain type of data. The major difference is that a `Connection` graph types comes with a bit more complexity to handle the added complexity of pagination. Implementing a `Connection` in a query is not difficult at all:

```csharp
Connection<FriendType>()
  .Name("friends")
  .Bidirectional()
  .Argument<StringGraphType>("query", "The search query you want to use to filter friends")
  .ResolveAsync(async context => {
    // ToDo: Retrieve our data
  });
```

You can specify whether your connection is either bidirectional or unidirectional by using the `.Bidirectional()` or `.Unidirectional()` extension methods. Unidirectional pagination will lead to only the `first` and `after` arguments being exposed, while bidirectional pagination will expose the `first`, `after`, `last`, and `before` arguments for use on the connection.

> **Mind the gap:**  
> My most common error retrieving data for a GraphQL endpoint had to do with the returned type being incorrect. The `ResolveAsync` method requires an function which returns a `Task<object>` type as argument, so at compile time anything goes. Check if your function either:
> - Returns a `Task<Connection<Friend>>` if you use the `ResolveAsync` method without `async` modifier.
> - Returns a `Connection<Friend>` if you use the `ResolveAsync` method with `async` and `await` modifiers, or if you use the non-async `Resolve` method.
> 
> *\* Assuming the `Friend` type is your model of choice*.  
> *\* Go [here](https://github.com/graphql-dotnet/graphql-dotnet/blob/master/src/GraphQL/Types/Relay/DataObjects/Connection.cs) for the `Connection` class.*

## <a href="#data-retrieval" id="data-retrieval">6.</a> Data Retrieval
From here on it gets interesting. I follow the following generic structure for all the endpoints I implement. By more or less following this natural structure you create a predictability which makes it very pleasant to work with this code, and therefore makes it more maintainable.

1. Resolve the provided arguments
2. Application of the filter options to the query
3. Slicing of the data set for pagination results
4. Retrieval of the data
5. Creation of `Connection` object

### <a href="#data-models" id="data-models">6.1</a> *[Contextual]* Model and Graph Types

Before digging into some details about the data retrieval process, let's get a good understanding of the data models we're working with throughout the example.

First we have a POCO (Plain Old CLR Objects) which contains our data. You might map this to your favourite data source. Whether you fill this object with data from a SQL database, ElasticSearch instance, some local text file or you fill it by scraping some website, it doesn't matter. For this article we're working with SQL, though.

```csharp
public class Friend {
  public string Cursor { get; set; }
  public string Name { get; set; }
  public string City { get; set; }
}
```

The graph type for this model would look like this:

```csharp
public class FriendType : ObjectGraphType<Friend>
{
  public FriendType() {
    Field(friend => friend.Cursor);
    Field(friend => friend.Name);
    // true at the end indicates that friend.City is allowed to be null.
    Field(friend => friend.City, true);
  }
}
```

In order to add a bit of sense to the next steps we'll start using the [SqlKata](https://github.com/sqlkata/querybuilder) library to dynamically build our SQL query. SqlKata provides a QueryFactory which couples SqlKata with [Dapper](https://github.com/StackExchange/Dapper) and provides `Query` objects which connection objects have already been initialized. Therefore allowing direct query execution without having to worry about connection strings and so. This `QueryFactory` is registerd in the DI container, and requested from the specific graph type. Feel free to forget this instantly. It might be of interest to the people wanting to go this route.

> **Aside:**  
> *I figured that it is nearly impossible to get cursor based pagination to work with Entity Framework, based on the available api's. See [this feature request for `.SkipWhile()`](https://github.com/aspnet/EntityFrameworkCore/issues/14104) for some more background on this. Linked by [this issue about creating connections](https://github.com/graphql-dotnet/relay/issues/25), which is the inspiration for this article.*

### <a href="#ic_arguments" id="ic_arguments">6.2</a> Resolving Arguments
Arguments are being resolved as is done with all other GraphQL queries:

```csharp
var search = context.GetArgument<string>("search");
// ... and so on
```

If an argument is not assigned to in the query then the `context.GetArgument<T>("search")` method will return `null`.

For lists I have the approach where I do the assignment and null check together:

```csharp
var list = context.GetArgument<List<string>>("list") ?? new List<string>();
```

The worst that will happen is that some iteration will not happen anymore. It's quite easy to defend yourself to exeptions happening due to zero elements in the list.

### <a href="#data-filters" id="data-filters">6.3</a> Data Filters
Important to keep in the back of your head is that we are following [a structure](#data-retrieval) where we are only defining filters to our data, but not yet retrieving the data itself.

Over time I developed a habbit to put all filter and query logic related to a single model inside a single method. This method has an argument list describing all kinds of possible filter operations on an object. I stopped caring about what-if questions on data filtering. If the api's users decides to apply some obscure combination of filters on a query then they kind of deserve it to face, at worst, an empty list.

Only having optional filter arguments gives total flexibility when using this method to filter through your data. An important aspect is utilizing these arguments in a way that they are always optional. Resolving an argument happens as shown below. Only an if statement with a query mutation.

```csharp
// Note that unlike Entity Framework, SqlKata does not use pure functions
if (!string.IsNullOrWhiteSpace(search)) sqlQuery.Where(q => q
    .WhereContains("Name", search)
    .OrWhereContains("City", search));
```

> For a more concrete example, take a look at [section **7**](#realworldsample).

The beauty of this approach is that you have all possible operations in a single place, while all being incredibly maintainable. This is also a nice place to tuck away some filters which determine what data the user may access. This way it is possible to apply authorization rules at once, for a single type at a time.

An added bonus this approach makes it is way easier than before to build specialized and highly usable search interfaces (both within code and client apps). Ironically, by focussing on the way we build our endpoints and therefore enabling end users to quickly filter through data, we make the whole problem of pagination a little bit less important.

### <a href="#slicing" id="slicing">6.4</a> Slicing for Pagination

Because we haven't yet materialized our query we can apply some more filters which are strictly related to pagination. It's important to do this as the last step before retrieving your data. Failing to do so would result in unpredictable and strange resultsets.

Based on your data provider, these slicing operations might be one of the most difficult things to implement efficiently. [Given the implementation of these slicing operations are worth a post on itself I have described my process here](/blog/2019-03-06/cursor-based-pagination-with-sql-server). This post focusses on the generation of SQL statements from the four pagination arguments using C# and SqlKata.

 Key is to write something generic which can be used for once and for all (types). While the result might be a bit slower than otherwise potentially possible, the upside is that development time is reduced significantly. Last but not least is that this will result in consistency across your platform. Expensive functions can always removed or refactored later, after they start behaving like bottlenecks.

### <a href="#connection-object" id="connection-object">6.5</a> Creating the `Connection` object

Last but not least is the instantiation of the connection. To make my life easier I have created an extension method that does the heavy lifting for me. It essentially combines three tasks:

1. Application of the filter operations
2. Data retrieval
3. Transformation into a `Connection` object

Step three is the tricky part. Part of this step is determining whether there are any next or previous slices available. To get this to work we need to retrieve the total number of available records for the provided filters, and we need to know about the rownumbers of the records in our slice. Most of the logic is just wiring everything up together.

> *Some magic to make this code a bit easier to understand (and untangle) is to cast a `Query` to a `XQuery` object. The `XQuery` object is created when a query object is retrieved from the [`QueryFactory`](https://sqlkata.com/docs/execution/setup). It will prevent us from requiring a reference to the `QueryFactory` object itself.*

```csharp
public static async Task<Connection<T>> ToConnection<T, TSource>(
    this SqlKata.Query query,
    ResolveConnectionContext<TSource> context,
    string cursorField = "Id") 
    where T : class, IId
    where TSource : class
{
    var xQuery = query as XQuery;

    if (xQuery == null) throw new ArgumentException("Make sure the query object is instantiated from a queryFactory", nameof(query));

    var countQuery = xQuery.Clone();

    countQuery
        .Clauses
        .RemoveAll(q => q.Component == "select"
                        || q.Component == "order");

    var totalCount = (await countQuery
        .SelectRaw("COUNT(*)")
        .GetAsync<int>())
        .SingleOrDefault();

    if (totalCount == 0) return new Connection<T>();
            
    var statement = xQuery.Compiler.Compile(
        xQuery.Slice(
            context.After?.FromCursor().ToString(),
            context.First.GetValueOrDefault(0),
            context.Before?.FromCursor().ToString(),
            context.Last.GetValueOrDefault(0)));

    var dictionary = new Dictionary<long, T>(xQuery.Connection.Query(
        sql: statement.Sql,
        param: statement.NamedBindings,
        map: (T t, long i) => new KeyValuePair<long, T>(i, t),
        splitOn: "RowNumber"));

    if (!dictionary.Any()) return new Connection<T>();

    var connection = new Connection<T>
    {
        Edges = dictionary
            .Select(q => new Edge<T>
            {
                Node = q.Value,
                Cursor = q.Value?.Id.ToCursor()
            })
            .ToList(),
        TotalCount = totalCount,
        PageInfo = new PageInfo
        {
            HasPreviousPage = dictionary.First().Key > 1,
            HasNextPage = dictionary.Last().Key < totalCount
        }
    };

    connection.PageInfo.StartCursor = connection.Edges.First().Cursor;
    connection.PageInfo.EndCursor = connection.Edges.Last().Cursor;

    return connection;
}
```

Usage of this extension method is demonstrated in [**section 7.1**](#).

## <a href="#realworldsample" id="realworldsample">7.</a> Overview of a real-world implementation

In order to make a little bit more sense of this article and to visually check if this method is something for you to implement, I have added some interesting parts of one of my real-world implementations. The goal of this code is to act as reference if you get stuck somewhere and want to see what the bigger picture looks like.

### <a href="#realworldconnection" id="realworldconnection">7.1</a> The Connection

First of there's the connection itself. Loads of arguments and an authorization rule (I will cover authorization with GraphQL in-depth another time).

In my humble opinion I think this GraphQL connection looks amazing because of its simplicity. All this part does is define arguments on our GraphQL endpoint, resolve these arguments, and uses these arguments in our query function (about which I have written more in [section 4 (architecture)](#architecture)). 

```csharp
Connection<FlightType>()
    .Name("flights")
    .AuthorizeWith("read:flight")
    .Bidirectional()
    .Argument<ListGraphType<GuidGraphType>>("cursors",
        "The cursors of the flights to retrieve")
    .Argument<StringGraphType>("query",
        "The search query can be used to search for a relevant piece of information. Things you can search on include aircraft registrations and callsigns, but also airfield names, ICAO codes or IATA codes.")
    .Argument<ScopeGraphType>("scope",
        "The scope to search through for flights")
    .Argument<DateTimeGraphType>("timestamp", 
        "Retrieves all active flights at this given moment")
    .Argument<ListGraphType<StringGraphType>>("aircraft", "")
    .Argument<ListGraphType<StringGraphType>>("airfields", "")
    .Argument<DateTimeGraphType>("fromTime", "")
    .Argument<DateTimeGraphType>("toTime", "")
    .Argument<StringGraphType>("departureAirfield", "")
    .Argument<StringGraphType>("arrivalAirfield", "")
    .Argument<ListGraphType<FlightOrderType>>("order", "")
    .ResolveAsync(async context =>
    {
        var list = context
            .GetArgument<List<Guid>>("cursors")?
            .Where(q => q != null && q != Guid.Empty)
            ?? new List<Guid>();

        var aircraftList = context
            .GetArgument<List<string>>("aircraft")?
            .Where(q => !string.IsNullOrWhiteSpace(q))
            ?? new List<string>();

        var airfieldList = context
            .GetArgument<List<string>>("airfields")?
            .Where(q => !string.IsNullOrWhiteSpace(q))
            ?? new List<string>();

        var query = context.GetArgument<string>("query");
        var timestamp = context.GetArgument<DateTime>("timestamp");
        var fromTimestamp = context.GetArgument<DateTime>("fromTime");
        var toTimestamp = context.GetArgument<DateTime>("toTime");
        var fromAirfield = context.GetArgument<string>("departureAirfield");
        var toAirfield = context.GetArgument<string>("arrivalAirfield");
        var order = context.GetArgument<List<string>>("order");

        return await queryFactory.FlightsQuery(
            list,
            timestamp,
            fromTimestamp,
            toTimestamp,
            fromAirfield,
            toAirfield,
            order,
            aircraftList,
            airfieldList,
            query)
            .ToConnection<Flight, object>(queryFactory, context);
    });
```

### <a href="#realworlddataaccess" id="realworlddataaccess">7.2</a> Data access function

The `FlightsQuery` extension method is responsible for composing the query which describes how to retrieve our data. There's not much exciting going on in this method.

```csharp
public static Query FlightsQuery(
    this QueryFactory queryFactory,
    IEnumerable<Guid> list = default,
    DateTime timestamp = default,
    DateTime fromTimestamp = default,
    DateTime toTimestamp = default,
    string fromAirfield = default,
    string toAirfield = default,
    IEnumerable<string> order = default,
    IEnumerable<string> aircraftList = default,
    IEnumerable<string> airfieldList = default,
    string query = default)
{
    var sqlQuery = queryFactory.Query("Flights")
        .Join("Aircraft", "Aircraft.Id", "Flights.AircraftId")
        .LeftJoinAs("Airfields", "DepartureAirfield", "Flights.DepartureAirfieldId", "DepartureAirfield.Id")
        .LeftJoinAs("Airfields", "ArrivalAirfield", "Flights.ArrivalAirfieldId", "ArrivalAirfield.Id")
        .WhereNotNull("Aircraft.Registration");

    if (list != default && list.Any()) sqlQuery.WhereIn("Id", list);
    if (aircraftList != default && aircraftList.Any()) sqlQuery.WhereIn("Aircraft.Registration", aircraftList);
    if (airfieldList != default && airfieldList.Any()) sqlQuery.Where(q => q.WhereIn("DepartureAirfield.Icao", airfieldList).OrWhereIn("ArrivalAirfield.Icao", airfieldList));

    if (timestamp != default) sqlQuery.Where(q => q.Where("Flights.DepartureTime", ">", timestamp.ToString("s")).Where("Flights.ArrivalTime", "<", timestamp.ToString("s")));
    if (fromTimestamp != default) sqlQuery.Where(q => q.Where("Flights.DepartureTime", ">", fromTimestamp.ToString("s")).OrWhere("Flights.ArrivalTime", ">", fromTimestamp.ToString("s")));
    if (toTimestamp != default) sqlQuery.Where(q => q.Where("Flights.DepartureTime", "<", toTimestamp.ToString("s")).OrWhere("Flights.ArrivalTime", "<", toTimestamp.ToString("s")));

    if (!string.IsNullOrWhiteSpace(fromAirfield)) sqlQuery.WhereContains("DepartureAirfield.Icao", fromAirfield);
    if (!string.IsNullOrWhiteSpace(toAirfield)) sqlQuery.WhereContains("ArrivalAirfield.Icao", toAirfield);
            
    if (!string.IsNullOrWhiteSpace(query)) sqlQuery.Where(q => q
        .WhereContains("Aircraft.Registration", query)
        .OrWhereContains("Aircraft.Callsign", query);   // ... etc

    if (order.Any())
    {
        foreach (var o in order)
        {
            if (o.Contains("^")) sqlQuery.OrderBy(o.Replace("^", ""));
            else sqlQuery.OrderByDesc(o);
        }
    }

    return sqlQuery;
}
```

A common argument against this type of code is that the lines are long. And I do agree on that. While maintaining this code one usually has the tendency to check for the argument names, which are located in the if-statements at the beginning of the line. For me personally this technique does wonders for readability.

> In case you are wondering; the `LeftJoinAs` method is one I wrote myself for convenience, and is not included with SqlKata. It came from the need to join the same table multiple times. What it does:
> ```csharp
> public static Query LeftJoinAs(this Query query, string table, string alias, string first, string second, string op = "=")
> {
>     return query.LeftJoin(new Query(table).As(alias), q => q.On(first, second, op));
> }
> ```

### <a href="#order" id="order">7.3</a> Defining order on a connection

While I'm at it I might explain this one as well. Like you might have seen in the previous method I also give the users the option to give order to the final result. It is one of the functionalities which is almost a requirement to do cursor based pagination. There are two things I like to accomplish:

- Prevent the users from using any arbitrary string
- Give the users the power to intuitively discover the functionality of the api themselves

GraphQL provides an amazing (and also underdocumented feature) to achieve this. These are the `EnumerationGraphType` types. You can do two things with them:

- Use an enum type as argument in the api
- Provide a mapping between two strings (more on this)

The first one is really easy. You only have to define your type as follows (assuming `UserRole` is your enum):

```csharp
public class UserRoleType : EnumerationGraphType<UserRole> { }
```

The second method of defining an enum within GraphQL perfectly suits our use-case. Consider that the api and database both do not have a flat structure, but we want to query through them from the connection. In order to achieve this we would have to connect field names with (joined) column names. 

The solution is to use a field name which is recognizeable to the users of the api, but under water returns another string field. The `AddValue` method has three arguments: `string name, string description, object value`.

```csharp
public class FlightOrderType : EnumerationGraphType
{
    public FlightOrderType()
    {
        Name = "FlightField";
        
        // ^ indicates a field should be ordered ascending
        AddValue("DepartureTime", "", "^DepartureTime");
        AddValue("DepartureTimeDesc", "", "DepartureTime");
        AddValue("DepartureIcao", "", "^DepartureAirfield.Icao");
        AddValue("DepartureIcaoDesc", "", "DepartureAirfield.Icao");
        AddValue("ArrivalTime", "", "^ArrivalTime");
        AddValue("ArrivalTimeDesc", "", "ArrivalTime");
        AddValue("ArrivalIcao", "", "^ArrivalAirfield.Icao");
        AddValue("ArrivalIcaoDesc", "", "ArrivalAirfield.Icao");
        AddValue("Registration", "", "^Aircraft.Registration");
        AddValue("RegistrationDesc", "", "Aircraft.Registration");
        AddValue("Callsign", "", "^Aircraft.Callsign");
        AddValue("CallsignDesc", "", "Aircraft.Callsign");
    }
}
```

In order to have a simple way to order both ascending and descending I have used the `^` sign as indication. See [the logic in **7.2**](#realworlddataaccess) for more details on how this is resolved.

This is a method which works for me, and I did not see any reason to make it more complex than it currently is.

## <a href="#ic_other-resources" id="ic_other-resources">8.</a> Other Resources

- [**Cursor based pagination with SQL Server using C# (and SQL)**](/blog/2019-03-06/cursor-based-pagination-with-sql-server): *Post detailing implementing cursor based pagination from C# against SQL Server.*
- [**graphql-dataloader-connections-demo**](https://github.com/corstian/graphql-dataloader-connections-demo): *A demo project showing usage of connections, and usage of connections in combination with the data loader.*
- [**MemeEconomy.Insights**](https://github.com/corstian/MemeEconomy.Insights): *First of all, I wanted to prove a point that it is possible to use Reddit as a data source in a GraphQL api. [r/memeeconomy](https://reddit.com/r/memeeconomy) is a subreddit in which you can invest so called 'MemeCoins' on new and innovative memes. In order to visualise what memes are investment worthy I'm hooking in to a live Reddit feed to collect statistics on submissions. This information is later exposed via a GraphQL api. At the time of writing it is still a work in progress.*

