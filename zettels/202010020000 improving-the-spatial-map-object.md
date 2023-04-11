---
title: "Improving the spatial map object"
slug: "improving-the-spatial-map-object"
date: "2020-10-02"
summary: ""
references: 
  - '[[202007230000 high-performance-2d-radius-search]]'
  - '[[201902250000 representing-coordinates-in-a-human-readable-way]]'
---

#software-development

Some time ago I published a blog post in which I described an object with which elements could be stored in a 2D grid. The type which turned out to be the `SpatialMap<T>` worked quite well, and due to the binary searches used to locate elements reasonably fast. The biggest limitation of the data type however was the limitation that both the X and Y axis could only contain unique values. The implication of this was that if there were three points which were positioned as being a triangle while aligned on both the X and Y axis, only one or two of these could be inserted in the object, depending on the insertion order.


> Check out the repository over on GitHub ([Skyhop/SpatialMap](https://github.com/skyhop/SpatialMap)), or download from [NuGet](https://www.nuget.org/packages/Skyhop.SpatialMap).
> 
> You can find the original post over here: [High Performance 2D Radius Search](/blog/2020-07-23/high-performance-2d-radius-search)

![Depending on the insertion order only one point ([1,1]) or two points ([1,3] and [2,1] could be inserted)](/uploads/Spatial_Map_Insertion_8435e48ca7.png)


This specific issue turned out to be fairly problematic as there was a real possibility this would lead to unexpected behaviour down the line. As such I needed to find a solution which would make this object more reliable and predictable.


## Solving those issues

Even though I was initially thinking about randomizing my X/Y coordinates with a small number to prevent this problem, I already required 14 digits of the double to store the location precisely, so I only had two reliable digits left to randomize (I'm storing coordinates as kilometers from the Greenwich mean line and the equator to simplify the math a bit). This solution would possibly reduce the amount of discarded points, but introduced difficulties when trying to query specific points. Additionally, an estimate is that I had a 5% change of this object interfering with my analysis, so ultimately that change would be brought down to 0.05% at best. Personally I already have troubles chasing a 5% bug down, let alone a 0.05% bug, so ultimately I didn't go down that way.

What I did do however was to implement a feature to stack points on top of each other. Even though I do not yet know what the performance implications of this change are, I suspect they are relatively low because the seek functions are still optimized. Essentially I modified the object stored within the modified `SortedList` object to contain a `List<T>` instead of `T`.

## The updated logic;

Below I'll show all this code once again, as I did in last blog, and I'll mention some of the changes I've implemented;

### The base of it all;

Underneath there is a `CustomSortedList` which overrides some functionality from the `SortedList` ([the reference source](https://referencesource.microsoft.com/#mscorlib/system/collections/sortedlist.cs)). It depends on reflection to access some internal properties to speed up object retrieval.

Most important are the following two features;

1. Access an object within the `SortedList` by index instead of by key value.
2. Instead of looking up the index of an exact key, look up the index with the key which most closely resembles the looked up key.

This last method is a significant change from the previous version, as the `IndexOfKey` method was overridden. In the meantime I figured out that it would be beneficial to both be able to look up a rough index, as well as check whether a key existed as is, and to retrieve said index.

```csharp
internal class CustomSortedList<TKey, TValue> : SortedList<TKey, TValue>
    where TKey : notnull
{
    private readonly FieldInfo _keysField = typeof(CustomSortedList<TKey, TValue>).BaseType.GetField("keys", BindingFlags.Instance | BindingFlags.NonPublic);
    private readonly FieldInfo _valuesField = typeof(CustomSortedList<TKey, TValue>).BaseType.GetField("values", BindingFlags.Instance | BindingFlags.NonPublic);
    private readonly FieldInfo _comparerField = typeof(CustomSortedList<TKey, TValue>).BaseType.GetField("comparer", BindingFlags.Instance | BindingFlags.NonPublic);

    // Returns the index of the entry with a given key in this sorted list. The
    // key is located through a binary search, and thus the average execution
    // time of this method is proportional to Log2(size), where
    // size is the size of this sorted list. The returned value is -1 if
    // the given key does not occur in this sorted list. Null is an invalid 
    // key value.
    // 
    public int RoughIndexOfKey(TKey key)
    {
        if (key == null) throw new ArgumentNullException(nameof(key));

        int ret = Array.BinarySearch<TKey>(
            (TKey[])_keysField.GetValue(this),
            0,
            Count,
            key,
            (IComparer<TKey>)_comparerField.GetValue(this));

        return ret >= 0 ? ret : ~ret;
    }

    // Returns the value of the entry at the given index.
    // 
    public TValue GetByIndex(int index)
    {
        if (index < 0 || index > Count) return default;
        return ((TValue[])_valuesField.GetValue(this))[index];
    }
}
```

### Accessing a point by their coordinate

I find it a bit silly to post a 200 line code snippet on my blog for no apparent reason. If you would like to see all code involved, I have created a repository containing this object can be found over at [https://github.com/skyhop/SpatialMap](https://github.com/skyhop/SpatialMap).

The general logic stayed the same, with the change that I am dealing with a `List<T>` instead of `T`. This is reflected within the `Add` method, where I'll first check existence of a key, and based on that will add the new element;


```csharp
public void Add(T element)
{
    lock (_mutationLock)
    {
        var x = _xAccessor(element);
        var xBucket = _x.IndexOfKey(x);

        if (xBucket >= 0)
        {
            _x.GetByIndex(xBucket).Add(element);
        }
        else
        {
            _x.TryAdd(x, new List<T> { element });
        }

        var y = _yAccessor(element);
        var yBucket = _y.IndexOfKey(y);

        if (yBucket >= 0)
        {
            _y.GetByIndex(yBucket).Add(element);
        }
        else
        {
            _y.TryAdd(y, new List<T> { element });
        }
    }
}
```

This same mechanism happens when removing an object; first I'll check the count for a specific key, after which I'll determine the action to take;

```csharp
public void Remove(T element)
{
    if (element == null) return;

    lock (_mutationLock)
    {
        var x = _xAccessor(element);
        var xBucket = _x.IndexOfKey(x);

        if (xBucket >= 0 && _x.GetByIndex(xBucket).Count > 1)
        {
            _x.GetByIndex(xBucket)
                .Remove(element);
        }
        else
        {
            _x.Remove(x);
        }

        var y = _yAccessor(element);
        var yBucket = _y.IndexOfKey(y);

        if (yBucket >= 0 && _y.GetByIndex(yBucket).Count > 1)
        {
            _y.GetByIndex(yBucket)
                .Remove(element);
        }
        else
        {
            _y.Remove(y);
        }
    }
}
```

Any further changes have to do with reading a `List<T>` instead of an `T`, which practically involve a few additional foreach statements throughout the code, to tackle any potential additional number of elements.
