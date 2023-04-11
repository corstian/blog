---
title: "How to add JWT Bearer auth to GraphQL subscriptions on ASP.NET Core 2.1?"
slug: "asp-net-core-2-1-and-graphql-adding-jwt-bearer-validation-to-subscriptions"
date: "2018-09-10"
summary: ""
references: 
  - [[201903080000 implementing-pagination-with-graphql-net-and-relay]]
---

#software-development #dotnet #graphql

***Please note before reading: this post flows over from implementation details of the [graphql-dotnet](https://github.com/graphql-dotnet/) project. If you're like me, stuck on authorization for subscriptions, and want to know how I worked around it, read the post. If you just want authorization with subscriptions to work, copy past the code blocks.***

The folks working on the [graphql-dotnet](https://github.com/graphql-dotnet/graphql-dotnet) library have done some amazing work on bringing [the GraphQL specification](https://facebook.github.io/graphql/) to the .NET ecosystem. While the library is definitely not easy to implement, the resulting GraphQL API can be a delight to work with.

By now several additional libraries have been developed. Notoriously [the authorization library](https://github.com/graphql-dotnet/authorization). This library brings policies and permissions to your GraphQL API. This authorization library is not dependent on any specific authorization mechanism, as long as you can provide an implementation for the `IProvideClaimsPrincipal` interface which sole responsibility is providing GraphQL with a `ClaimsPrincipal` instance. Validation logic is provided by implementing the `IValidationRule` interface.

While this works fine for queries and mutations, at the time of writing the following issues with regards to making bearer authorization work on subscriptions:

* HTTP headers are provided in the payload of the first frame that goes over the WebSocket connection. This first frame has the type of `connection_init`. As the subscription subscriber is only invoked at the `start` frame, it might seem difficult at first sight to retrieve the JWT token from the payload of the `connection_init` frame.

> If you want to know what is going on within a websocket connection, the developer tools inside browsers can show the individual frames on websocket connections these days, which is a nice way of getting to know some about the protocols flowing through :)

* Subscriptions hijack the `UserContext` property on the `ResolveEventStream` object to pass a `MessageHandlingContext` instance. Therefore, generic auth validation logic provided with an `IValidationRule` implementation cannot access the ClaimsPrincipal object on the `UserContext`, therefore failing verification.

## Retrieving the bearer token

While the GraphQL library looks pretty daunting at first sight, it also happens to be incredibly extensible at all points through the use of interfaces. One of these extension points happens to be the `IOperationMessageListener`, which acts on different messages received via the websocket connections, and therefore, indirectly on subscriptions. I have implemented the `IOperationMessageListener` in a way that the bearer token can be extracted from the `connection_init` frame.

```csharp
public class SubscriptionInitializer : IOperationMessageListener
{
    private IHttpContextAccessor _contextAccessor;
    public SubscriptionInitializer(IHttpContextAccessor httpContextAccessor)
    {
        _contextAccessor = httpContextAccessor;
    }

    public async Task BeforeHandleAsync(MessageHandlingContext context)
    {
        if (context.Terminated) return;

        var message = context.Message;

        if (message.Type == MessageType.GQL_CONNECTION_INIT)
        {
            JwtSecurityTokenHandler handler = new JwtSecurityTokenHandler();
            var user = handler.ValidateToken(message.Payload.GetValue("Authorization").ToString().Replace("Bearer ", ""),
                new TokenValidationParameters
                {
                    ValidIssuer = "TODO: Enter your issuer",
                    ValidAudience = "TODO: Enter your audience",
                    IssuerSigningKey = new JsonWebKey(@"TODO: Load your JWKS")
                },
                out SecurityToken token);

            if (!user.Identity.IsAuthenticated) await context.Terminate();

            _contextAccessor.HttpContext.User = user;
        }
    }

    public Task HandleAsync(MessageHandlingContext context)
    {
        return Task.CompletedTask;
    }

    public Task AfterHandleAsync(MessageHandlingContext context)
    {
        return Task.CompletedTask;
    }
}
```

## Passing on the IPrincipal

As seen in the code above it is not too difficult to validate a JWT for a subscription, but what if we're using the authorization library and want our existing `IValidationRule` implementations also to apply for subscriptions?

To understand one of the possible ways we can pass this data to the `IValidationRule` implementations we have to dig into the DI system. The GraphQL library heavily relies on the DI mechanics, and the only thing we want to know is how `IValidationRule` objects are passed, or in this case, injected.

You don't have to figure that out yourself. A type of `IEnumerable<IValidationRule>` is injected into the constructor of the `DefaultGraphQLExecuter<TSchema>` which is being registered in the DI as transient, therefore meaning that it is instantiated every time it is requested. Thankfully the `IGraphQLExecuter<TSchema>` is requested in the `GraphQLHttpMiddleware<TSchema>` middleware, so with every request we get a new executer. Great!

Because of these properties we can create our own object which is injected through the constructors both in the `IOperationMessageListener` and `IValidationRule` implementations. We will transfer the principal by means of this transport object. We can then populate the `ClaimsPrincipal` in the `UserContext` with the value we passed in this terrible, awefull, and just plain ugly way, but at least it's better than duplicating code multiple times.

Now here comes the exciting part (As inspired by [this commented out class](https://github.com/graphql-dotnet/server/blob/develop/samples/Samples.Server/AddAuthenticator.cs), which I only discovered after I figured everything else out…). We can inject the `IHttpContextAccessor` into our class to have the ability to get the current `HttpContext` instance into our `SubscriptionInitializer`. Even more beautifull is the way we can pass the `ClaimsPrincipal` to the `IValidationRule`: We set the `HttpContext.User` property with our `ClaimsPrincipal`, after which it is available on the current `HttpContext`. How easy can life be sometimes.

Also, don't forget to inject the `IHttpContextAccessor` into your `IValidationRule` in order to be able to access the `IPrincipal`.
