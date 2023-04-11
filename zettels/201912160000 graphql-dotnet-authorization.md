
---
title: "GraphQL.NET: Authorization"
slug: "graphql-dotnet-authorization"
date: "2019-12-16"
summary: ""
---

#software-development #dotnet #graphql

> ***The implementation described in this article is available on [GitHub](https://github.com/corstian/Boerman.GraphQL.Contrib), and a package can be downloaded through [NuGet](https://www.nuget.org/packages/Boerman.GraphQL.Contrib/2.0.0-preview1).***

One of the topics related to building a GraphQL API which I have tried to postpone as much as possible is authorization. There are multiple reasons for this. If I implement authorization I would like to do it correctly, in a way that the whole API is correctly secured, and in a way which makes sense related to industry standards. Given that I'd rather be lazy (and prevent errors due to my own stupidity) I prefer to use existing packages to handle most of the authorization logic.

Even though there are existing packages which deal with authorizing queries built with the `graphql-dotnet` package, I haven't found one yet which is clearly documented, and whose methods of authorizing queries lies in line with the code-first conventions used with the `graphql-dotnet` package.


## An currently existing alternative

The solution I seemed to be able to get kind of working was the policy based approach as provided with the `GraphQL.Server.Authorization` namespace (available in the `GraphQL.Server` package). With this method you define several policies within the service collection, so you can reuse these by name from an endpoint, which may look as follows:

```csharp
Field<StringGraphType>()
    .Name("name")
    .AuthorizeWith("policyName");
```
 Whereas the policies themselves may be registered as follows:

```csharp
// Instead of the.AddGraphQLAuthorization() call:
services.AddHttpContextAccessor()
    .AddTransient<IValidationRule, AuthorizationValidationRule>()
    .AddAuthorizationCore(_ =>
    {
        _.AddPolicy("", p => p
            .RequireAuthenticatedUser()
            .RequireClaim("claim", "claimValue"));
    });
```

 Though this is a framework native approach to authorization, it does not fit well with me - nor do I feel this fits within the GraphQL code-first philosophy - that I have to create an authorization policy first, which later has to be referenced from an endpoint.

Note that the .AddGraphQLAuthorization call does not work correctly with ASP.NET Core 3.0, due to a deprecation of the `AddAuthorization` method.


## My goals

In my environment I wish to authorize to an API using an JWT bearer token. The way 

In order to stay in line with the (fluent) code-first conventions for building a graph API, my aim is to be able to define the authorization policies right on the field definitions. This way the authorization rules would be grouped together with the resolver, and gives the ability to quickly figure out what the authorization requirements are for a certain `GraphType`.

Even though this adds a a little bit of verbosity to the code, the end result will be much more clearer as it prevents you from hopping through different parts of your projects to collect the different bits of required information, hence resulting in a lower cognitive load.

From a performance point of view the additional authorization metadata will not be a problem either. Most, if not all `GraphTypes` are instantiated as a singleton instance either way.


## The implementation

To implement authorization we'd need two different components. One middleware, which verifies whether the authorization policies evaluate successfully, and an extension method with which we can add authorization policies to the metadata dictionary of a specific `GraphType`.

I don't necessarily (feel) like rewriting the authorization systems already available through ASP.NET, and therefore I will use the `AuthorizationPolicy` type which is available through the `Microsoft.AspNetCore.Authorization` package.

> This implementation will make some assumptions about your environment. It's expected that you are running the `graphql-dotnet` library on ASP.NET Core (3.1), and that you're using the [`GraphQL.Server.Transports.AspNetCore`](https://www.nuget.org/packages/GraphQL.Server.Transports.AspNetCore/) package as middleware.


### Defining the rules

The `Microsoft.AspNetCore.Authorization` package also contains the `AuthorizationPolicyBuilder`, which can be used to define all the authorization rules we wish to apply. 

> As to validate these rules later on, the `IServiceCollection.AddAuthorizationCore()` extension method will add an `IAuthorizationService` implementation instance to the DI container. This class contains the `AuthorizeAsync` which will verify whether the current `ClaimsPrincipal` has the access rights as defined in the `AuthorizationPolicy`.

To add the authorization rules to the metadata dictionary we'll add an extension method. To be quite honest, this is the least exciting, yet most interesting part of the authorization implementation, because it's so simple, yet so versatile. The base method:

```csharp
public static void WithPolicy(
    this IProvideMetadata type,
    Action<AuthorizationPolicyBuilder> policy)
{
    var policyBuilder = new AuthorizationPolicyBuilder();

    policy.Invoke(policyBuilder);

    var authorizationPolicy = type.GetMetadata(PolicyKey, policyBuilder.Build());

    type.Metadata[PolicyKey] = authorizationPolicy;
}
```

Note that this method already runs the `AuthorizationPolicyBuilder` so that only the actual `AuthorizationPolicy` is put in the metadata dictionary.

There are extension methods for the different builders:

```csharp
public static FieldBuilder<TSourceType, TReturnType> WithPolicy<TSourceType, TReturnType>(
    this FieldBuilder<TSourceType, TReturnType> builder,
    Action<AuthorizationPolicyBuilder> policy)
{
    builder.FieldType.WithPolicy(policy);
    
    return builder;
}

// In case you're using Connections :)
public static ConnectionBuilder<TSourceType> WithPolicy<TSourceType>(
    this ConnectionBuilder<TSourceType> builder,
    Action<AuthorizationPolicyBuilder> policy)
{
    builder.FieldType.WithPolicy(policy);

    return builder;
}
```

The way these extension methods can be used is as follows:

```csharp
// METAR stands for "Meteorological Aerodrome Report"
Field<Metar>()
    .Name("metar")
    .WithPolicy(policy => policy
        .RequireAuthenticatedUser()
        .RequireAnyScope("all:weather"))
    .Resolve(c => new { });
```


### Validating the authorization policy

The middleware will materialize in the form of an `IValidationRule`, similar to what is implemented in the `GraphQL.Server` project. See https://github.com/graphql-dotnet/server/blob/develop/src/Authorization.AspNetCore/AuthorizationValidationRule.cs for an example. This file is adapted to our needs for validating authorization policies.

If you like to know about the internals when it comes to validating the policy I strongly suggest you check the adaption on GitHub: [https://github.com/corstian/Boerman.GraphQL.Contrib/blob/master/Boerman.GraphQL.Contrib/PolicyValidationRule.cs](https://github.com/corstian/Boerman.GraphQL.Contrib/blob/master/Boerman.GraphQL.Contrib/PolicyValidationRule.cs). Summarized it will take the metadata of the `GraphType` to be resolved, check whether it has an `AuthorizationPolicy`, and if so, validate these against the current `ClaimsPrincipal`.

#### How about subscriptions?

This method works well with `query` and `mutation` queries, but not so much with `subscription` queries. This is because subscriptions use a somewhat different method for authorization. The way this has been resolved is by adding an `IOperationMessageListener` implementation which extracts the bearer token from an initialization message on the websocket connection, and uses this as the `ClaimsPrincipal` for the request. For further details about this method can be found in the source file [here](https://github.com/corstian/Boerman.GraphQL.Contrib/blob/master/Boerman.GraphQL.Contrib/SubscriptionPrincipalInitializer.cs).

## Usage

Please note that the implementation is targeted towards (at the time of writing) alpha packages for the GraphQL project, targeting ASP.NET Core 3.0. These packages can be found on the GraphQL MyGet feed at `https://www.myget.org/F/graphql-dotnet/api/v2/package`.

First download the NuGet package: [https://www.nuget.org/packages/Boerman.GraphQL.Contrib/2.0.0-preview1](https://www.nuget.org/packages/Boerman.GraphQL.Contrib/2.0.0-preview1). The graph type authorization policy helpers are first implemented in the `2.0.0-preview1` package.

In order to register the middleware which validates the `AuthorizationPolicy` within `Startup.cs`:

```csharp
services.AddGraphQL(options =>
    {
        options.EnableMetrics = true;
        options.ExposeExceptions = true;
    })
    .AddPolicyValidation()
```
In order to define authorization policies on a `GraphType`, use the `WithPolicy` extension method:

```csharp
Field<ProfileType>()
    .Name("me")
    .WithPolicy(policy => policy
        .RequireAuthenticatedUser()
        .RequireAnyScope("personal:profile"))
    .ResolveAsync(async context => throw new NotImplementedException());
```

> Please note that the [`Boerman.GraphQL.Contrib`](https://github.com/corstian/Boerman.GraphQL.Contrib) package is my own personal testbed for features which are useful to me, and therefore I can't guarantee there will not be any breaking changes in this package. If you depend on it I suggest you either fork the repository, or you watch it for any activity so that you can stay somewhat up to date.

