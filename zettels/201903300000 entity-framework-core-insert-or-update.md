---
title: "Entity Framework Core: insert or update, the lazy way"
slug: "entity-framework-core-insert-or-update"
date: "2019-03-30"
summary: ""
---

#software-development #dotnet #data-storage

It seems as if I'm getting more lazy by the day. That's great, because I didn't really feel like manually mapping my data to my data models in order to have Entity Framework update them.

The thing is there are several options for you if you wish to run such operation:

- Merge: This operation combines two data sets into one based on primary keys. The [ZZZ Entity Framework Extensions library](https://entityframework-extensions.net/bulk-merge) has some nice methods to help you do this, although proprietary.
- Upsert: Upsert is a legit term used in some database engines to indicate an operation which either updates or inserts data. I for one had never heard about that one before. The [`FlexLabs.Upsert`](https://github.com/artiomchi/FlexLabs.Upsert) is really nice for some operations, but did not quite fit my needs. More on that later.
- Or you can do the old try to retrieve an entry, if it exists, update is, and if it doesn't, then create a new one. But this didn't really fit my needs either.

As I said before. I'm lazy. Real lazy. Let me explain my situation.

I've been fiddling around quite a lot with GraphQL lately. During this process I figured out that the create and update logic I write on the client-side is quite simillar. This led me to think that, unless a special operation needed to be executed, most create and update logic could be combined in `save`, or in my case `mutate` operation. Depending on whether the primary key would be provided either an update or create operation should be executed.

So all in all I did not feel like writing stuff like this:

```csharp
dbModel.FirstName = viewModel.FirstName;
dbModel.LastName = viewModel.LastName;
// etc...
```

Heck, I even ditched use of the viewmodels, mostly. I wanted to have an operation with which I could just pass on my object, and it would automagically determine which fields needed to be sent to the database.

Something like this is what I came up with. I discourage using this directly in production code. Use it by means of inspiration.

```csharp
public static async Task Mutate<T>(
    this DbSet<T> dbSet,
    T subject,
    Guid? cursor = default) where T : class, IId
{
    var context = dbSet.GetService<ICurrentDbContext>().Context;

    bool entryExists = false;

    if (cursor != default) entryExists = await dbSet.AsNoTracking().AnyAsync(q => q.Id == cursor);

    if (entryExists == false) {
        subject.Id = cursor ?? Guid.NewGuid();
        dbSet.Add(subject);
    }
    else if (entryExists == true)
    {
        subject.Id = cursor.Value;

        var entry = dbSet.Attach(subject);

        entry.State = EntityState.Modified;

        typeof(T)
            .GetProperties()
            .Where(q =>
                q.CanWrite
                && Convert.GetTypeCode(q.GetValue(subject)) != TypeCode.Object
                && q.GetValue(subject) != q.GetType().GetDefault()
                && q.GetValue(subject) != null)
            .Select(q => q.Name)
            .ToList()
            .ForEach(property => entry.Property(property).IsModified = true);
                
        entry.Property(q => q.Id).IsModified = false;
    }
}

public static object GetDefault(this Type type)
{
    return type.IsValueType
        ? Activator.CreateInstance(type)
        : null;
}

public interface IId {
  Guid Id { get; set; }
}
```

It's not as magical as it might look like. Although there are a few things to note:

- There's a check for `CanWrite` as I had a readonly property which was ruining this idea
- The `TypeCode` comparison is to remove any objects, and therefore also lists.
- It is important to compare a value with it's default value. Checking for null only will not work well with most value types.

I know there are certain people which are going to scream out loud about performance. Let me state this: performance was not a priority, nor did I benchmark it. If this ever becomes an issue I will improve it. If it doesn't, well, then that's great.

Happy hacking! And remember, life's too short to keep writing CRUD code.

