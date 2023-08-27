---
title: "Stream processing SQL-stored data in C#"
slug: "stream-processing-sql-stored-data-in-c"
date: "2018-11-07"
summary: ""
references: 
---

#software-development #dotnet #sql

Recently I had a brainfart that told me to retroactively process about 2 billion data points. The kind of processing that had to be done was to run all these data points (describing flights) through an algorithm that extracted some metadata.

First and foremost my problem is this; the data is stored in a MSSQL instance, whereas this processing algorithm is written with C#. Now these two generally play fairly nice with each other, but in this case, with these amounts of data (+- 1TB) you just can't load it all into memory, and process it all at once. Another option would be to use batching, but I'd be too lazy (and it'd be too slow) to use.

## Why not use Entity Framework?

Entity Framework is a great ORM! I absolutely love it for my day to day work and I wouldn't want to work without it. However, Entity Framework has it's limitations, too. One of these is that Entity Framework retrieves the complete dataset, before it returns. What we want to achieve is that we receive part of the data, already before the SQL query has finished executing.

## Let's get coding

The gist of it is easy. Open a connection to the server, execute a command, and retrieve the results.

> There's really not too much to this code. This is more of a reference for I know future me will hate previous me if I do not make this note.

```csharp
using (var connection = new SqlConnection("[connectionstring]"))
using (var command = new SqlCommand("[query]", connection))
{
    await connection.OpenAsync();

    using (var reader = await command.ExecuteReaderAsync()) {
        if (reader.HasRows) {
            while (await reader.ReadAsync()) {

                reader.GetInt32(0);
                // Or use any applicable cast for the data type you're trying to retrieve
            }
        }
    }

    connection.Close();
}
```

## The aftermath

In the end this code ran for about 26 hours to crunch 'n munch through all this data. If you care about performance, take this from me, and do not run this on your local machine. I have ran the .NET code directly on the database machine for network bandwidth and dev machine uptime not to be a limiting factor.
