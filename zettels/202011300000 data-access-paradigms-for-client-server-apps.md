
---
title: "Data access paradigms for client/server apps"
slug: "data-access-paradigms-for-client-server-apps"
date: "2020-11-30"
summary: ""
---

#software-development

A few days ago I found this question on Reddit in which someone asked about whether to put any data processing logic on the client side or on the server side of an application. For someone starting out with web development, or just simply any generic structure in which data is pulled from a server, choices like this can be daunting. Additionally, the results of picking one choice over the other are not always immediately clear. Over the course of several years I have made an incredible amount of bad choices with regard to front and backend structuring, which has led me to an understanding of some best practices which will make your life significantly easier, and therefore this post;


## Context

The advice/best practices I'm describing in this post are primarily applicable in any situation where a backend and frontend is used. I'm not exactly sure whether some of this advice will be applicable to serverless solutions as well, as the 'backend' is sometimes entirely nonexistent (think about firebase etc.).

## Technology choices

Personally I do not care about the technology you use to create a certain amount of communication between the backend and the frontend. However, what is important to realise is that the less restrictive this communication channel is, the more flexibility this gives on your frontend. As such I have settled with GraphQL (for now).

## Frontend code reuseability

For maintainability purposes, putting any data processing code on the backend is the ultimate reuseability trick for sharing across multiple clients.

What I regard as an even more important aspect of this is that offloading data processing to the server will give you more leeway with regard to the architectural rigidness required on the client side. When done correctly, the client side may simply act solely as a secured box which contains the views to properly display the data. It will make the client significantly easier to maintain.

> Consider an alternative, where processing logic is spread over the server and the client. When there is a certain aspect which needs maitenance, you'll spend a significant amount of energy in determining where the piece of code you need is located in the first place. By having to navigate both the server project as well as the client project, the cognitive energy you'll expend is significantly higher.

One of the greates simple ideas I ever had was to calculate the UNIX timestamp I needed for use in a charting library on the back end. Simply not needing to calculate this every time I wanted to build a chart saved me so much time and cognitive agony that I think it's been one of the greatest shift in mindset for me personally.

## Back end structure

On the back end I can highly recommend to offer a single entry point for each data model you offer to the client side. This single access point should contain all possible filter and query options applicable to this model. The entry point on the API should on itself be just a mapping between the API parameters, and a method which constructs the query for said data model. [See this for a real world example of mine where I implement this approach.](/blog/2019-03-08/implementing-pagination-with-graphql-net-and-relay#realworlddataaccess)

> Again, the alternative to this approach is creating a method which will query a very specific thing based on very specific parameters. You will end with a dozen query methods for each data model, which is incredibly hard - if not impossible - to maintain. Having a generalized data access method (though frowned upon by some) will enable you to move more quickly, while still being allowed to create a custom data access method when needed, be it for query complexity, performance, or other reasons.
> This on itself is a shift in mindset, whereas you'll move to a generic first method. Just let the computer do what it's good at. You can always improve upon that later.
>
> While I am at it, some completely unsubstantiated claims about generic data access methods are that they are slow and untestable. Writing very specific queries will make you slow, and when that 5% performance improvement is valued more than your salary, you're either severely underpaid or you're in the 1% of developers whom do not need to read this post. Additionally, testability is as good as you write the contents of this data access method. See the previous link for a construction method which yields predictable results.

## Mental model

Another important aspect of streamlining communication between client and server is the modelling of your data models and interaction with the API after business processes, rather than technical processes. Having a data model which reflects the business process makes it significantly easier to continue development. It'll prevent you from thinking about the translation between business process and technical implementation every step on the way.

## Reasons to deviate

Doing all data processing on the server is no magic bullet whatsoever. There are situations in which it might be more appropriate to do a select amount of data processing on the client side. One example of such would be continuous recomputations of existing data, which for performance reasons might be better to do directly on the client.

## What are your ideas?

The topics I covered in this post have all been inspired by my experience, and I believe many other people have many different experiences. If you would like to share your paradigm shifts, please do! I'd be delighted to read those :)

