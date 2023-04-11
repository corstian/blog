
---
title: "Asynchronous streams in C# and scrolling through ElasticSearch with NEST"
slug: "asynchronous-streams-in-csharp-and-scrolling-through-elasticsearch-with-nest"
date: "2020-03-12"
summary: ""
---

#software-development #dotnet #data-storage

## Introduction

ElasticSearch is an amazing piece of technology to store a shitload of pieces of information. For my use case (with [Skyhop](https://skyhop.org)), I require this information to be processed before it is slightly usable. Most of the time this processing step happens even before the data enter my ElasticSearch cluster, however what is a data analysis system worth when you're unable to reprocess data again to correct errors, or apply new insights?

## Problems
In order to process information I went on to solve these two primary issues:

1. How do you manage to query a month worth of data without overloading the available memory resources (remember, sometimes the maximum query size of 10_000 documents only represents a period of 3 seconds)
2. How to retrieve more than 10_000 documents with NEST (the ElasticSearch .NET client)?

I have the benefit of the whole data processing happening asynchronously. This prevents further complications which arise in storing and sorting information, as I can just shovel it into the processing layer and let it do its magic.

## Code snippets! 😎
Actually this is one of the neater pieces of code I have written, thankfully to [C# 8.0's asynchronous streams](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams) which enable me to shovel data back as soon as it's available. Refer to the linked article for more in depth background on how it's being used.

As for my code, I am using the dependency container to inject an `ElasticClient` instance into the class. Wire this up as suits your situation.

```csharp
public class ElasticQueries {
    private readonly ElasticClient _elasticClient;

    public ElasticQueries(ElasticClient elasticClient) {
        _elasticClient = elasticClient;
    }

    public async IAsyncEnumerable<QueryType> Query(
        DateTime startTime,
        DateTime endTime)
    {
        if (startTime == null
            || endTime == null)
        {
            yield break;
        }

        var result = await _elasticClient.SearchAsync<QueryType>(q => q
            .Index("queryIndex")
            .Query(w => w
                .Bool(e => e
                    .Filter(r => r
                        .DateRange(t => t
                            .Field(y => y.Timestamp)
                            .GreaterThanOrEquals(startTime)
                            .LessThanOrEquals(endTime)
                        )
                    )
                )
            )
            .Size(10_000)
            .Scroll("10s")
            .Sort(w => w.Ascending(e => e.Timestamp))
        );

        while (result.Documents.Any()) {
          foreach (var document in results.Documents) {
            yield return document;
          }

          result = await _elasticClient.ScrollAsync<QueryType>("10s", result.ScrollId);
        }
    }
}
```

In the code above there are a few distinctive sections:

1. Parameter checks
2. Query definition and initiation
3. Looping over results and scrolling further


Now while we can access the data we have to use a special foreach method in order to retrieve it. The neat thing is that every time a scroll request returns its data we can access it directly wherever we need it.

```csharp
// In my case _queries is injected via the DI container and would be of type `ElasticQueries`
await foreach (var document in _queries.Query(DateTime.UtcNow, DateTime.UtcNow.AddDays(-1)))
{
    // Voila, access the document and do whatever it pleases you here :)
}
```

And that's all there is to that.

