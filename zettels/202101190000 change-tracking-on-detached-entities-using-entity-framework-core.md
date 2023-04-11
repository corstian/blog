
---
title: "Change tracking on detached entities using Entity Framework Core"
slug: "change-tracking-on-detached-entities-using-entity-framework-core"
date: "2021-01-19"
summary: ""
---

#software-development #dotnet

There are a surprising amount of use cases where it is beneficial to deal with disconnected entities in EF core. In my particular situation I'm dealing with a domain which is responsible for modifying data through an API. Reasonably enough, by the time this data returns to the server, the original `DbContext` instance with which this data had been retrieved is no longer around, and therefore there is no change tracking.

EF Core tries to go out of the way, and most of the time it's fairly easy to hook the entities back to the context, but with a single exception; when removing entities. Though this is not a problem when directly telling the context to remove a specific element, it is when the object contains a collection from which one of the children needs to be removed. Removing an object from a collection removes it without trace, and there's no way Entity Framework may know there has ever been an object in there without verifying it with the database.


[This article describes several ways to deal with disconnected entities.](https://docs.microsoft.com/en-us/ef/core/saving/disconnected-entities) Succinctly summarised the following is going on;

1. When using auto generated keys, an empty primary key identifies a newly created object, while a populated primary key identifies an updated object.
2. When using non-auto generated keys you're on your own.
3. If auto generated keys are used, `context.Update` can be used for both object creation as wel for updates.
4. Having a graph which contains new as well as updated objects, AND auto generated keys, you can simply call the `context.Update` method.
5. There are three approaches to deleting entities suggested;
    1. A soft delete by flipping a bit
    2. Directly deleting an object by passing it to the context
    3. Retrieving the object from the database to compare it with the local version.

Personally I prefer not to have to worry about maintaining local state, while on the other hand I do not feel comfortable pulling an object from the server once again to compare states.

Additionally, Julie Lerman suggested local state management is the way to go. ([See this](https://stackoverflow.com/a/19060945/1720761), [and this](https://thedatafarm.com/data-access/more-on-disconnected-entity-framework/), [and this](https://docs.microsoft.com/en-us/archive/msdn-magazine/2016/august/data-points-ef-core-change-tracking-behavior-unchanged-modified-and-added).)

## A compromise
Because I'm only updating data from my domain, and most of the stuff is categorised within aggregates, I only have to keep track of children within the boundaries of these aggregates. In the end the only compromise is that I'll have a slightly different method to mark an entity as being deleted, and a `BaseRepository` instance which knows how to deal with these. Here's a writeup about my approach;

### Requirement 1; two interfaces
In order to have a way to speak to entities of mine, I want to know where I can find information about their state;

```csharp
public interface ITrackChanges
{
    [NotMapped]
    public bool IsMarkedForRemoval { get; }

    internal void MarkForRemoval();
}
```

The second object is an interface which describes a basic repository to be able to deal with entities;

```csharp
public interface IRepository<T, TKey>
{
    Task<T> ById(TKey id);

    Task<T> Create(T obj);
    Task<T> Update(T obj);
    Task<T> Remove(TKey id);
}
```

### Requirement 2; an entity

I try to prevent direct access to properties as much as possible. It improves maintainability, and prevents some person (most likely myself, as I work alone most of the time) from directly changing the properties and therefore possibly putting the data model into an invalid state. Most of the model is not interesting, that is, besides the logic by which I access child collections, and the way I remove a child entity.

```csharp
public class Group : Entity<Guid>
{
    private Group() { }

    public Group(string name)
    {
        Name = name;

        _profiles = new List<GroupProfile>();
    }

    public string Name { get; private set; }

    private List<GroupProfile>? _profiles = null;
    public IEnumerable<GroupProfile> Profiles => _profiles?.Where(q => !q.IsMarkedForRemoval);
    
    public void RemoveProfile(Guid profile) {
        _profiles.FirstOrDefault(q => q.ProfileId == profile).MarkForRemoval();
    }
}
```

Outside of this `Group` no one really has access to the `List` itself, that is, if you're not using reflection. There is an intentional choice as to why I'm not executing a `ToList()` call after the `Where` clause, but more about that later. This way I'm able to act as if I have removed an object, while preserving it locally.

### Requirement 3; a generic repository implementation
I hate writing the same code twice, which is why I created the following repository from which I derive my actual repositories;

```csharp
public class RepositoryBase<T, TKey> : IRepository<T, TKey> where T : class
{
    private readonly Func<DbContext> _service;
    private readonly Func<DbContext, IQueryable<T>> _accessor;
    private readonly Expression<Func<T, TKey>> _expression;

    public RepositoryBase(
        Func<DbContext> service,
        Func<DbContext, IQueryable<T>> accessor,
        Expression<Func<T, TKey>> expression)
    {
        _service = service;
        _accessor = accessor;
        _expression = expression;
    }

    private Expression<Func<T, bool>> _createExpression(TKey value)
    {
        return Expression.Lambda<Func<T, bool>>(
            Expression.Equal(
                _expression.Body,
                Expression.Constant(value)
            ),
            _expression.Parameters);
    }

    public async Task<Result<T>> ById(TKey id)
    {
        var query = _accessor.Invoke(_service());
        return await query.FirstOrDefaultAsync(_createExpression(id));
    }
    
    public async Task<T> Create(T obj)
    {
        var query = _accessor.Invoke(_service());

        var db = _service();
        db.Set<T>().Add(obj);
        await db.SaveChangesAsync();

        return obj;
    }

    public async Task<T> Remove(TKey id)
    {
        var query = _accessor.Invoke(_service());
        var obj = await query.FirstOrDefaultAsync(_createExpression(id));

        if (obj == null) throw new Exception($"Could not remove {typeof(T).Name} with id {id}");

        var db = _service();
        db.Set<T>().Remove(obj);
        await db.SaveChangesAsync();

        return obj;
    }

    public async Task<T> Update(T obj)
    {
        var query = _accessor.Invoke(_service());

        var db = _service();

        var navigationProperties = obj
            .GetType()
            .GetProperties()
            .Where(p =>
            (
                typeof(IEnumerable).IsAssignableFrom(p.PropertyType)
                && p.PropertyType != typeof(string)
            ) || p.PropertyType.Namespace == obj.GetType().Namespace);

        foreach (var navigationProperty in navigationProperties)
        {
            if (navigationProperty.GetValue(obj) is ITrackChanges prop
                && prop.IsMarkedForRemoval)
            {
                db.Entry(obj).State = EntityState.Deleted;
            }
            else if (navigationProperty.GetValue(obj) is IEnumerable<ITrackChanges> enumerable)
            {
                var source = enumerable.GetType()
                    .GetField("_source", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance)
                    .GetValue(enumerable) as IEnumerable<ITrackChanges>;

                foreach (var item in source.Where(q => q.IsMarkedForRemoval).ToList())
                {
                    db.Entry(item).State = EntityState.Deleted;
                }
            } 
        }

        db.Set<T>().Update(obj);
        await db.SaveChangesAsync();
        return obj;
    }
}
```

> **Fair warning:** The code above crashes when a navigation property is not an `IQueryable` type. Oh the joys of reflection.

As I already wrote, I'm lazy, and if it is not possible I prefer to keep my hands of tedious operations such as database mutations. Much of the logic above is boilerplate which I have included for sake of clarity. What I want to focus on is the way I update an object, and the role the `ITrackChanges` interface has therein. The reason for building this piece of crap is because I'm torn up between two considerations;

1. I want to hide the removed objects from any type dealing with the aggregate.
2. I want to be able to retrieve the removed element without having to make it available.

If I were to keep the removed object widely available, and have the expectation all dependent code checks themselves whether a child is due for removal, I can be sure this will eventually go wrong somewhere. Besides that, it would start leaking business logic, which is not what I want. Using the current approach it is possible to execute a type cast to a `List` or `IQueryable` object, though that's something I consider to be just shy of using reflection. As of now I believe this is the best solution, for the time being.

> **P.S.** Know any better solution? Please leave a comment. Looking forward to finding alternative solutions!

