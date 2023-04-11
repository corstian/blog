
---
title: "Cursor based pagination with C# and SQL Server"
slug: "cursor-based-pagination-with-c-and-sql-server"
date: "2018-12-26"
summary: ""
---

#software-development #dotnet #sql #graphql

> **I have written a newer post containing a better approach to doing cursor based pagination. [Check the updated post here!](/blog/2019-03-06/cursor-based-pagination-with-sql-server)**  
> This post describes the first baby steps I had to make to get to the final result. While it may contain some interesting information to you, it isn't as flexible as the alternative approach I came up with later. Check it out!

---

## Introduction

Out of pagination techniques, cursor based pagination is the one that allows for the most flexible and easy implementation, in my humble opinion. [See this post for an explanation how Slack evolved to cursor based pagination.](https://slack.engineering/evolving-api-pagination-at-slack-1c1f644f8e12)

The beautiful thing about cursor based pagination is that it gets you a whole lot of benefits:

1. You do not have to retrieve the total number of items (which is quite costly)
2. Your pagination method plays well with live data
3. It forces you to think about a more user friendly way to implement pagination. (Mainly thinking about search and filter options).
4. This way of implementing pagination allows for a flexible implementation at the client side, based on UX related requirements.

If you read between the code, [this post](https://dzone.com/articles/why-most-programmers-get-pagination-wrong) does a pretty great job explaining why offset based pagination is bad, both from a user experience based perspective, as well as from a performance based point of view.

However, developer in the .NET space are not yet quite used to cursor based pagination. Why would they? `.Skip(page*10).Take(10)` works quite well for most use cases, after all.

## Goals

First of all we would need a way to prove we can efficiently implement this pagination method at all. But before we dive into the details we're going to define a set of rules to which this pagination method must adhere:

1. A cursor can be anything, from an identifier to (an in my eyes cleaner method) a base64 encoded value to identify unique items.
2. Order should be maintained for int, string and datetime properties
3. Live data shall not impact the resulting data (in most cases)

Essentially we should be able to order the result set on a specific field, and then retrieve a certain amount of data, relative to the specified cursor.

The cursor in question however, might be the item from the data source relative to which data after this one is being retrieved, or from which data before this point is being retrieved. Usually provided with the 'after' or 'before' parameters.

One of the primary difficulties is, that we should not order based on the identifier. Personally I prefer to use GUID's as cursor, as I can translate this directly to the primary keys as used in my data source. Besides this users cannot be expected to guess random entries, which is an additional barrier to take in case actual security measures lack, or are poorly implemented. No order can be inferred from GUID's, and possibly other opaque cursors, so we should not assume there is one.

## A proof-of-concept

This first example has been created to figure out whether it's possible, at all, to implement cursors, using Entity Framework 6 (Because LinqPad). In this example we treat the ID (78) as the cursor.

```csharp
var serialNumber = Devices.Where(q => q.Id == 78).Select(q => q.SerialNumber).FirstOrDefault();

Devices
	.OrderBy(q => q.SerialNumber)
	.Where(q => String.Compare(q.SerialNumber, serialNumber) >= 0)
	.Take(50)
	.Dump();
```

Surprisingly this works pretty well. Entity Framework perfectly translates the `String.Compare` method to SQL code. The generated SQL code is surprisingly clean.

```sql
SELECT TOP (1)
    [Extent1].[SerialNumber] AS [SerialNumber]
    FROM [dbo].[Devices] AS [Extent1]
    WHERE 78 = [Extent1].[Id]
GO

-- Region Parameters
DECLARE @p__linq__0 NVarChar(1000) = ''
-- EndRegion
SELECT TOP (50)
    [Project1].[Id] AS [Id],
    [Project1].[SerialNumber] AS [SerialNumber]
    -- And some more fields
    FROM ( SELECT
        [Extent1].[Id] AS [Id],
        [Extent1].[SerialNumber] AS [SerialNumber],
        -- Again, some more fields here
        FROM [dbo].[Devices] AS [Extent1]
        WHERE [Extent1].[SerialNumber] >= @p__linq__0
    )  AS [Project1]
    ORDER BY [Project1].[SerialNumber] ASC
```

This query would be composed with a cursor defined as after: 78. Implementing a before: 78 operation would require us to use an inverted sorting method. No big deal.

## Properly handling queries

This is something, when done right, that would be repeated many, many times over and over inside your application. The cost of implementing pagination should be neglectable in order to promote an uniform API design. Given we have proven the effectiveness of cursor based pagination, we now have to tackle the following questions:

* How to dynamically order the results based on a provided field?
* How to implement the before/after cursor behaviour?
* How to pour this into a reusable component?

There's not much to help us with this in Entity Framework (Core). I would have imagined that we could use the \`SkipWhile()\` method for that. We'd use it like \`Table.SkipWhile(q =&gt; q.Id &lt; 1684).Take(10);\`. However, \`SkipWhile\` is not implemented (yet). I set on to look for other possibilities. While considering to use Dapper for this functionality in the time being I wasn't really keen on writing and validating my own SQL query builders so I went to look for sql builders. I ended up with SqlKata. This library seems to be a nice middle way between using Entity Framework and manually writing your queries. After all, code that has not been written is code that doesn't need to be tested.

## Implementation

After all it only took about 15 lines of code to achieve the wanted behaviour with SqlKata in an extension method. When used at the end of your query (so the `Query` object contains all your other clauses), this extension methods compiles the from and orderby clauses in order to be consistent with the order of the result set to the application.

```csharp
/// <summary>
/// A generic method for applying cursor based pagination.
/// </summary>
/// <param name="query">The query to apply pagination to</param>
/// <param name="column">The column to use for cursor based pagination</param>
/// <param name="cursor">The cursor itself</param>
/// <param name="count">The number of items to retrieve after the cursor</param>
/// <returns></returns>
public static Query CursoredOffset(this Query query, string column, string cursor, int count)
{
    var compiler = new SqlServerCompiler();

    var ctx = new SqlResult
    {
        Query = query
    };

    var from = compiler.CompileFrom(ctx);
    var order = compiler.CompileOrders(ctx);

    query = query.CombineRaw($@"OFFSET (
        SELECT TOP 1 rn
            FROM(
                SELECT {column}, ROW_NUMBER() OVER({ order }) AS rn { from }
            ) T
            WHERE T.{column} = '{cursor}'
        ) rows
        FETCH NEXT {count} ROWS ONLY");

    return query;
}
```

**What's next?** I'm going to publish a library containing some extension methods for use with SqlKata which enables you to easily use cursors in your queries.

Happy coding!
