
---
title: "GraphQL API Design: Cursor"
slug: "graphql-api-design-cursors"
date: "2019-10-24"
summary: ""
---

#software-development #dotnet #graphql

One of the favourite API design techniques I have applied a lot lately is to use so called "cursors" throughout API's to improve its consistency.

Summarized; use cursors when:

- You mix multiple different types of identifiers throughout your API (strings, integers, guid's)
- You do not want to imply order when there is none

Opposite, feel free to ignore cursors when:

- You consistently apply a single identifier type throughout the whole API/backend, and do not see that changing in the future.
- You do not care about API consistency


## Why?

To get one thing out of the way; I do not believe having different types of identifiers is necessarily an indicator for bad software design. I think it's a good practice to keep the identifier type consistent in a single data source, but besides that there's nothing wrong with mixing multiple data stores in an application. This probably is one of the use cases in which GraphQL is exceptionally suited to get the job done.

My inspiration from using cursors originally stems from a little line in the GraphQL documentation found at [https://graphql.org/learn/pagination/](https://graphql.org/learn/pagination/). Here it states:

> "As a reminder that the cursors are opaque and that their format should not be relied upon, we suggest base64 encoding them." - [https://graphql.org/learn/pagination/](https://graphql.org/learn/pagination/)

Thus the great thing is that the cursor is just some base64 encoded data, and therefore most data types can easily be encoded. [In the past I have written helper functions which help converting several data types to a base64 string.](https://github.com/corstian/Boerman.GraphQL.Contrib/blob/master/Boerman.GraphQL.Contrib/Cursor.Extensions.cs) Though trivial code, I believe it's important to consciously apply cursors to maintain a consistent API surface.

The mental model behind using cursors however is way more powerful than the technology itself. The aim of a cursor is not to secure the data contained by it, but rather to abstract it away, and as a result they simplify the mental model developers working with the API can employ.


## How?

Converting data to it's base64 representation is not too difficult. Several samples of how it's done with C# using extension methods:

```csharp
// Converting data types to it's base64 representation:

// A guid:
public static string ToCursor(this Guid guid) => Convert.ToBase64String(guid.ToByteArray());

// A string:
public static string ToCursor(this string str) => Convert.ToBase64String(Encoding.UTF8.GetBytes(str));

// An integer:
public static string ToCursor(this int i) => Convert.ToBase64String(BitConverter.GetBytes(i));

// Bytes:
public static string ToCursor(this byte[] bytes) => Convert.ToBase64String(bytes);


// Converting these same data types back, from their base64 representation:

// To a guid:
public static Guid FromCursorToGuid(this string base64) => new Guid(Convert.FromBase64String(base64));

// To a string:
public static string FromCursorToString(this string base64) => Encoding.UTF8.GetString(Convert.FromBase64String(base64));

// To an integer:
public static int FromCursorToInt(this string base64) => BitConverter.ToInt32(Convert.FromBase64String(base64), 0);

// To bytes:
public static byte[] FromCursorToBytes(this string base64) => Convert.FromBase64String(base64);
```

*This code can be found [on GitHub](https://github.com/corstian/Boerman.GraphQL.Contrib/blob/master/Boerman.GraphQL.Contrib/Cursor.Extensions.cs), and an package containing these extension methods is available [on NuGet](https://www.nuget.org/packages/Boerman.GraphQL.Contrib/).*

While it's straightforward to convert data to a cursor, it's more difficult to convert it back. Information on the original data type is lost when data is converted to it's base64 representation, and one needs to know about the data type the cursor is meant to represent.

This could possibly, and if someone puts in the work, be resolved by adding a little metadata on the data type to the cursor. And though it's possible, such approach is currently not worth the complexity for me. Besides that it would add a little overhead to the cursor, even though that is the least of my worries for now. 😅
