---
title: "Using the object capability security model in C# CRUD applications"
slug: "using-the-object-capability-security-model-in-crud-applications"
date: "2023-01-22"
summary: "This post shows an example about how object capabilities can be used in a CRUD application to defer authorization to individual models."
---

#software-development #dotnet #web-development #security

Within the field of software development, [object capabilities](https://en.wikipedia.org/wiki/Object-capability_model) (OCAP) are an architectural approach to securing subcomponents of software and giving these software components the authority to determine whether one can or cannot evaluate a given operation. This generally works by having the calling code supply a piece of identifying information about the principal (calling party, whether that's a machine or a human) based upon which it can determine whether the caller is authorized to execute said operation.

This approach to authorization is truly amazing for locality of code changes. By placing the authorization rules in close proximity with the actual behaviour we'll gain the following benefits:
- Authorization logic can be specifically tailored to the operation that is executed.
- The authorization rules cannot globally be changed.
- The rules can explicitly be tested for their correctness in conjunction with the actual behaviour.
It is with this increased locality of authorization rules, and therefore increased distribution, that the code base as a whole becomes more resilient to changes to authorization rules, and therefore becomes increasingly secure.

## Example of OCAP on CRUD models
As a simple demonstration about how object capabilities work we can start with a simple CRUD model. The beauty of a CRUD model is that it can be generalized for the four basic operations, being create, read, update and delete. In this demonstration we will not start defining the information contained in the model, but rather the abstraction to deal with the CRUD models themselves. Such abstraction might look like this:

```csharp
public interface ICrudHandler<T>
    where T : class
{
    public void Create(T model);
    public T Read(Guid id);
    public void Update(T model);
    public void Delete(T model);
}
```

In this interface we have just defined the behaviour which we expect to be available for all CRUD models. Using this abstraction we should be able to disconnect the data from the behaviour, simply by implementing behaviour separately from the data itself. To make it work we can simply supply an instance of CRUD model `T` to an implementation of the `ICrudHandler`.

It's at this point of creating generalizations that the object capability model comes in. There are two considerations we need to be aware of;
1. Authorization rules must be handled consistently for all models
2. Authorization logic is specific to the model instance itself, and should therefore be implemented on a per-model basis.

A simple abstraction with object capabilities can look like this, where the goal is to have a per-instance validation rule defining whether or not the principal can have access to the resource;

```csharp
public interface IPrincipal {
    /*
     * The IPrincipal interface is already one which exists in the .NET framework
     * so I do not recommend on using this one unless you know what you're doing.
     * 
     * Instead it is to represent authorizing information upon which you want to
     * make the decision whether or not to allow one to access the information.
     * 
     * This can therefore take any shape you want it to take.
     */

    public Guid UserId { get; }
}

public interface ICrudModel {
    public Func<bool, IPrincipal> MayAccess { get; } 
}

public interface ICrudHandler<T>
    where T : ICrudModel
{
    public void Create(T model);
    public T Read(Guid id);
    public void Update(T model);
    public void Delete(T model);
}
```

In this example the `ICrudModel` required an additional `AuthorizationRule` property, which functions as a way to validate access to the resource. Implementation on an user model looks like this [^1];

```csharp
public class User : ICrudModel {
    public Func<bool, IPrincipal> MayAccess = 
        (principal) => principal.UserId == this.UserId;

    public Guid UserId { get; init; }
}
```

To generalize authorization checks we must implement these alongside the CRUD operations themselves in an `ICrudHandler` implementation;

```csharp
public class CrudHandler<T> : ICrudHandler {
    private readonly IPrincipal _principal;
    
    public CrudHandler(IPrincipal principal) {
        _principal = principal;
    }
    
    public void Create(T model) {
        if (!model.MayAccess(principal)) throw new Exception("Unauthorized");
        
        // ... db logic to update database
    }

    public T Read(Guid id) {
        // ... db logic to retrieve model with given id

        if (!model.MayAccess(principal)) throw new Exception("Unauthorized");
    }

    public void Update(T model) {
        if (!model.MayAccess(principal)) throw new Exception("Unauthorized");
        
        // ... db logic to update provided model in db
    }

    public void Delete(T model) {
        if (!model.MayAccess(principal)) throw new Exception("Unauthorized");
        
        // ... db logic to delete entry from db
    }
}
```
[^2]

After seeing this approach one may decide that it is an ugly practice to embed authorization information on the data model, but to me such practice makes perfect sense. After all the sort of authorization that appropriately protects the data is highly dependent on the data that is stored. At the same time, some of these authorization rules may only be evaluated after the data in the model is known, thus providing an additional reason to combine some static information with the data itself. After all the goal is the protection of the data contained within the model, and the further away this protection is placed from the data, the bigger the vulnerabilities become. The general rule "out of sight; out of mind" applies here.

### (Testing for) security concerns
It is quite unfortunate for us that the strength of a security model depends upon its weakest link. In this case it holds the assumption that the authorization rules are checked from the CRUD handler, an assumption which only makes sense if the CRUD handler is the sole way data is retrieved. Therefore the security model becomes riddled with holes if one decides to bypass the CRUD handler, and directly query the database for information. One must therefore assure that dealing with the CRUD handler is easier than directly querying the database, and that this abstraction is at the same time flexible enough to aid in all use-cases. Consolidating such approach in the overall architecture helps as well. In case of a CRUD application this means creating generalized API handlers directly interfacing to the CRUD handler.

A benefit of this approach however is that we can test for security concerns. We can subject individual models to rigorous tests involving their authorization rules to see whether these line up with our assumptions.

```csharp
[Fact]
public void User_Can_Only_Access_Own_Data() {
    var guid = Guid.NewGuid();
    var user = new User() { Id = guid };

    var principal = new Principal(guid);

    Assert.True(user.MayAccess(principal));
}

[Fact]
public void User_Cannot_Access_Other_Data() {
    var user = new User() { Id = Guid.NewGuid() };
    var principal = new Principal(Guid.NewGuid());
    
    Assert.False(user.MayAccess(principal));
}
```

In addition to testing the authorization rules on the models, we can also test CRUD handlers to see whether they properly defer authorization checks to the models themselves.

These tests will prove to be an improvement to the overall security model, for they break when a change somehow impacts the security model. This means that if not caught locally, the CI/CD pipeline will break, therefore preventing such vulnerability from shipping to production. Never just trust a single layer of security to catch all attacks and vulnerabilities.

## In summary
- Generalize behaviour and couple it with authorization checking to have a consistent and predictable approach to security
- Defer authorization rules to individual models impacted in the operation to ensure the correct rules are applied
- Test both models and generalized behaviour for their security properties, and to add an additional layer of protection against vulnerabilities
- Reduce the amount of code assuming proper authorization had been acquired to a bare minimum. Therefore; do authorization checks close to the data, not on the edges of the system.

[^1]: In real-world situations the security cases are not always as clear cut as they were in this example. The information provided to such authorization rule may therefore be more detailed, and include more information such as the organizations to which an user belongs, roles and functions they have and more.
[^2]: To validate access rules during a read operation it is necessary that we first retrieve the model itself from the data store before we can check the access rules. After all we can still decide not to return the result. In a real-world situation however one may decide to implement a more complex form of authorization where these rules are checked to a certain extent before a roundtrip to the database.
