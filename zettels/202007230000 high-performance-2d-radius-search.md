---
title: "High performance 2D radius search"
slug: "high-performance-2d-radius-search"
date: "2020-07-23"
summary: ""
---

#software-development #dotnet #algorithms

> **UPDATE**
> In the first version of this post I have made an error with regard of the retrieval of possible matches. The update has fixed that.

> **Additional update;** I have made a few additions to this library, about which you can read [here](/blog/2020-10-02/improving-the-spatial-map-object). The library is now available on [GitHub](https://github.com/skyhop/SpatialMap), as well as [NuGet](https://www.nuget.org/packages/Skyhop.SpatialMap).

Sometimes I find myself in a situation where I need a very specific data structure with very specific properties. This time I needed something with which I can quickly retrieve nearby elements in 2D space. Although there are data structures perfectly suited to do so, such as a R-Tree, Kd-Tree and other related structures, I needed something which is very easy and somewhat performant to insert into and remove from.


As my only requirement is to look up nearby elements in 2D space, I choose to maintain two "index" lists which would hold the X and Y coordinates. These indices would be updated upon insertion and deletion.

## Storing coordinates

During my initial search for a solution I started using an `SortedDictionary` and tried using the `Array.BinarySearch` method to quickly identify the indices of the elements which contained the lower and upper bounds of the axis. Some performance tuning sessions later I discovered the conversion from an `SortedDictionary` to an `Array` type was too resource intensive. With a runtime of 70 ms over 100_000 elements I would be able to cut 2/3rds of the runtime off.

Some search queries later I found the `SortedList` classes, which offer an `IndexOfKey` method. The underlying logic also relies on a binary search, though the downside is that it required an exact match of the key. Thankfully we're able to modify this behaviour. The `Array.BinarySearch` method returns the index on an exact match, and `~index` in case there is no exact match. Using some reflection magic we're able to change this behaviour for our custom implementation.

```csharp
internal class CustomSortedList<TKey, TValue> : SortedList<TKey, TValue>
{
    private readonly FieldInfo keysField = typeof(CustomSortedList<TKey, TValue>).BaseType.GetField("keys", BindingFlags.Instance | BindingFlags.NonPublic);
    private readonly FieldInfo comparerField = typeof(CustomSortedList<TKey, TValue>).BaseType.GetField("comparer", BindingFlags.Instance | BindingFlags.NonPublic);
    private readonly MethodInfo getByIndexMethod = typeof(CustomSortedList<TKey, TValue>).BaseType.GetMethod("GetByIndex", BindingFlags.Instance | BindingFlags.NonPublic);

    // Returns the index of the entry with a given key in this sorted list. The
    // key is located through a binary search, and thus the average execution
    // time of this method is proportional to Log2(size), where
    // size is the size of this sorted list. The returned value is -1 if
    // the given key does not occur in this sorted list. Null is an invalid 
    // key value.
    // 
    public new int IndexOfKey(TKey key)
    {
        if (key == null) throw new ArgumentNullException(nameof(key));

        int ret = Array.BinarySearch<TKey>(
            (TKey[])keysField.GetValue(this),
            0,
            Count, 
            key,
            (IComparer<TKey>)comparerField.GetValue(this));

        return ret >= 0 ? ret : ~ret;
    }

    // Returns the value of the entry at the given index.
    // 
    public TValue GetByIndex(int index)
    {
        return (TValue)getByIndexMethod.Invoke(this, new object[] { index });
    }
}
```

This modification is important, because it allows us to find an non-exact match which enables us to retrieve the lower and upper indices reflecting the values we want to retrieve.

The `GetByIndex` method is one which is already on the `SortedList` object, but it's private. Because it's something I would love to use, I used some reflection magic to be able to access it.


## Implementing the SpatialMap object

Now that the most important problems are solved, we can get back to our class which has to filter nearby objects. The implementation essentially consists of two of these `CustomSortedList` instances which keep track of the axis values, and some methods to add, remove and query this object;

```csharp
public class SpatialMap<T>
{
    private readonly object _mutationLock = new object();
    private readonly Func<T, double> _xAccessor;
    private readonly Func<T, double> _yAccessor;

    public SpatialMap(
        Func<T, double> xAccessor,
        Func<T, double> yAccessor)
    {
        _xAccessor = xAccessor;
        _yAccessor = yAccessor;
    }
    
    private CustomSortedList<double, T> _x { get; set; } = new CustomSortedList<double, T>();
    private CustomSortedList<double, T> _y { get; set; } = new CustomSortedList<double, T>();

    public void Add(T element)
    {
        lock (_mutationLock)
        {
            if (!_x.TryAdd(_xAccessor(element), element))
            {
                return;
            }

            if (!_y.TryAdd(_yAccessor(element), element))
            {
                _x.Remove(_xAccessor(element));
            }
        }
    }

    public void Remove(T element)
    {
        lock (_mutationLock)
        {
            _x.Remove(_xAccessor(element));
            _y.Remove(_yAccessor(element));
        }
    }

    public IEnumerable<T> Nearby(T element, double distance)
    {
        var x = _xAccessor(element);
        var y = _yAccessor(element);

        var innerDistance = Math.Sqrt(Math.Pow(distance, 2) / 2);

        var lowerXIndex = _x.IndexOfKey(x - innerDistance);
        var upperXIndex = _x.IndexOfKey(x + innerDistance);
        var lowerYIndex = _y.IndexOfKey(y - innerDistance);
        var upperYIndex = _y.IndexOfKey(y + innerDistance);

        var hashSet = new HashSet<T>();
        
        var i = lowerXIndex;

        while (i < upperXIndex)
        {
            var el = _x.GetByIndex(i);
            if (hashSet.Add(el) 
                && Distance(
                    x, y,
                    _xAccessor(el),
                    _yAccessor(el)) < distance)
            {
                    yield return el;
            }

            i++;
        }

        i = lowerYIndex;

        while(i < upperYIndex - 1)
        {
            var el = _y.GetByIndex(i);
            if (hashSet.Add(el)
                && Distance(
                    x, y,
                    _xAccessor(el),
                    _yAccessor(el)) < distance)
            {
                yield return el;
            }

            i++;
        }
    }
}
``` 

An important detail is that the `Nearby` function creates an `HashSet` to keep track of the elements which have already been returned. Therefore it might or might not be important to implement a custom `GetHashCode` method on the objects you're using this method with.

The methodology of the `Nearby` function first retrieves all potential matches from a 'box', after which is determines whether the point is really within a specified distance. To calculate the distance we're simply using the Euclidean distance as follows;

```csharp
public static double Distance(double x1, double y1, double x2, double y2)
{
    var x = x2 - x1;
    var y = y2 - y1;

    return Math.Sqrt(x * x + y * y);
}
```

### Search optimization
In order to reduce the possible matches from the array sweep we're limiting the search area even more than the original radius. The reason for doing so is that with an distance of 20 on the X axis we will already exhausting all possible matches, in which case the second while loop would be totally obsolete, as is the `HashSet`. Instead we will be limiting the search area a little bit such that all possible matches will be retrieved only if both `SortedList`s are consulted. According to the pythagorean theorem for triangles with a 90° with legs `a`, `b` and `c`; <code>a<sup>2</sup> + b<sup>2</sup> = c<sup>2</sup></code>. If we would not change the maximum search distance, the maximum leg length for a maximum distance of 20 would be; <code>20<sup>2</sup> + 20<sup>2</sup> = sqrt(800)</code> which is roughly `28.28` at an angle of 45°. Given we only would want a maximum length of 20 at 45° we should compute our maximum values first.

This computation can be done by `sqrt(distance * distance / 2)`. With a distance of 20 the result would be roughly `14.14`. The difference in search area is `28.28 - 14.14 = 14.14`. We're literally cutting the search width in half, which is something with profound results on the search speed!


## Benchmarks

While anyone can claim an algorithm to be performant, it's much more interesting when there are actual benchmarks. I have seeded an `SpatialMap` instance with 100_000 data points randomly placed on an 2000x2000 grid. Effectively this means there is one point per 40 square units.

```
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.18363.959 (1909/November2018Update/19H2)
Intel Core i7-8650U CPU 1.90GHz (Kaby Lake R), 1 CPU, 8 logical and 4 physical cores
.NET Core SDK=3.1.302
  [Host]     : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT
  DefaultJob : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT


|             Method | Distance |        Mean |     Error |     StdDev |      Median |
|------------------- |--------- |------------:|----------:|-----------:|------------:|
| FindNearbyElements |        1 |    26.46 us |  1.229 us |   3.624 us |    26.63 us |
| FindNearbyElements |        2 |    47.00 us |  0.913 us |   0.937 us |    46.99 us |
| FindNearbyElements |        5 |   111.25 us |  1.781 us |   1.390 us |   111.65 us |
| FindNearbyElements |       10 |   276.34 us | 18.179 us |  53.600 us |   248.99 us |
| FindNearbyElements |       25 |   589.62 us | 42.437 us | 123.789 us |   552.93 us |
| FindNearbyElements |       50 | 1,034.05 us | 48.089 us | 141.790 us |   953.51 us |
| FindNearbyElements |      100 | 2,044.97 us | 87.147 us | 254.211 us | 1,949.99 us |
```

The expected number of points retrieved is as follows:

- Search distance of **1**; area of **3.14**; expected number of points **0.08**
- Search distance of **2**; area of **12.57**; expected number of points **0.31**
- Search distance of **5**; area of **78.54**; expected number of points **1.96**
- Search distance of **10**; area of **314.16**; expected number of points **7.85**
- Search distance of **25**; area of **1963.50**; expected number of points **49.09**
- Search distance of **50**; area of **7853.98**; expected number of points **196.35**
- Search distance of **100**; area of **31415.93**; expected number of points **785.40**

## Possible application

In order to speed up spatial calculations on real world data I have allowed myself to introduce certain amounts of error. One of these is by reflecting the latitudal and longitudal coordinates as an XY grid which reflects the number of kilometers from the Greenwich mean line as well as the Equator. Imagining the earth as flat coordinate system makes it much easier to work with distances, even though it isn't as accurate as possible. [I have written about this simplification over here.](/blog/2020-07-03/projecting-latitude-and-longitude-onto-a-flat-grid)
