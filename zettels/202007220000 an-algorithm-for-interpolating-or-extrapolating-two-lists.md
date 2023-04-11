
---
title: "An algorithm for interpolating or extrapolating two lists"
slug: "an-algorithm-for-interpolating-or-extrapolating-two-lists"
date: "2020-07-22"
summary: "Having my brain eat itself while trying to interpolate two data sets to one another I came up with this working, though overly complicated solution. It works, but can be implemented in a much less convoluted mess."
---

#software-development #dotnet #algorithms

During my journey in implementing the algorithms as described in the paper "High-performance spatiotemporal trajectory matching across heterogeneous data sources" I found a requirement to run linear interpolation on two data sources. In this blog post I describe the most important pieces of logic used for the interpolation or extrapolation of two sets. This code is written using C#, but the algorithm should be fairly easy to port to other languages.


## Data preparation

In order to make my life easier I have created a data structure to encapsulate two data points and a common timestamp. Based on this information one should be able to create an interpolation. The data structure in which I store this information is called the `InterpolationContainer`, for lack of a better name.

```csharp
public class InterpolationContainer<T>
{
    public long Time { get; set; }
    public T T1 { get; set; }
    public T T2 { get; set; }
}
```

When interpolating multiple data sets I first populate a list with all the data points and their timestamps. When iterating over this list I can easily spot which objects (either `T1` or `T2`) are still empty and need interpolation.

```csharp
var combination = new List<InterpolationContainer<T>>();

combination.AddRange(T1.Select(t => new InterpolationContainer<T>
{
    Time = getTime.Invoke(t),
    T1 = t,
    T2 = default
}));

combination.AddRange(T2.Select(t => new InterpolationContainer<T>
{
    Time = getTime.Invoke(t),
    T1 = default,
    T2 = t
}));

combination = combination
    .OrderBy(q => q.Time)
    .ToList();
```
*Wherein `T1` and `T2` are respectively both of type `List<T>` in the sample above. `getTime` is of type `Func<T, long>` and used to get a timestamp from `T`.*


## Determining points to interpolate between

The current implementation of the interpolation algorithm, or rather the algorithm which decides which point to iterate from, is quite sloppy. It is called as follows;

```csharp
for (var i = 0; i < combination.Count - 1; i++)
{
    combination[i].T1 = InterpolateSet(combination, interpolation, q => q.T1, i);
    combination[i].T2 = InterpolateSet(combination, interpolation, q => q.T2, i);
}
```

While the `InterpolateSet` method is as follows;

```csharp
private static T InterpolateSet<T>(
    List<InterpolationContainer<T>> list,
    Func<T, T, long, T> interpolation,
    Func<InterpolationContainer<T>, T> valueAccessor, 
    int i)
    where T : class
{
    if (valueAccessor.Invoke(list[i]) != null) return valueAccessor.Invoke(list[i]);

    int before = i == 0 ? 0 : i - 1;
    int after = i + 1;

    // Find a point before the point we want to interpolate
    while (before > 0)
    {
        if (valueAccessor.Invoke(list[before]) != null) break;
        before--;
    }

    // Find a point after the point we want to interpolate
    while (after < (list.Count - 1))
    {
        if (valueAccessor.Invoke(list[after]) != null) break;
        after++;
    }

    if (valueAccessor.Invoke(list[before]) == null && valueAccessor.Invoke(list[after]) == null)
    {
        throw new Exception("Not enough data");
    }

    // If no point in from of our interpolation target can be found, switch over to extrapolation
    // Start looking for two consecutive points after our current element
    if (valueAccessor.Invoke(list[before]) == null)
    {
        before = after;
        after++;
        while (after < list.Count)
        {
            if (valueAccessor.Invoke(list[after]) != null) break;
            after++;
        }
    }
    // Alternatively, if no point behind our interpolation target can be found, start looking for
    // two points in front of our target to extrapolate from
    else if (valueAccessor.Invoke(list[after]) == null)
    {
        after = before;
        before--;

        while (before > 0)
        {
            if (valueAccessor.Invoke(list[before]) != null) break;
            before--;
        }
    }

    return interpolation.Invoke(valueAccessor.Invoke(list[before]), valueAccessor.Invoke(list[after]), list[i].Time);
}
```

Globally the code above does the following things;

1. Check if there is a point in front of the point to interpolate
2. Check if there is a point after the point to interpolate
3. If no point in front of the target is found, go find two points behind the interpolation target
4. If no point behind the target can be found, go find two points in front of the interpolation target

## Interpolation logic

One interesting parameter which I have not yet mentioned is the `interpolation` parameter, which contains the user supplied interpolation logic. As the signature is `Func<T, T, long, T>` this method requires two objects of T to be supplied, from which an interpolation at a certain timestamp (as supplied by the long value) can be created. The return value should be an instance of `T` which represents the interpolated point.

With this method it is possible to dynamically supply the intra or extrapolation algorithm. Currently a linear interpolation between points is good enough, but in the future I might want to swap it out and calculate a more accurate estimation based on the points provided. Currently the logic to interpolate two sets is as follows;

```csharp
var interpolationResult = Interpolation.Interpolate(
    set1,
    set2,
    q => q.Timestamp.Ticks, // Timestamp is an property on `set1` and `set2`
    (object1, object2, time) =>
    {
        var dX = object2.X - object1.X;
        var dY = object2.Y - object1.Y;
        var dT = (object2.Timestamp - object1.Timestamp).Ticks;

        if (dT == 0) return null;

        double factor = (time - object1.Timestamp.Ticks) / (double)dT;

        return new Record
        {
            Timestamp = new DateTime((long)(object1.Timestamp.Ticks + dT * factor)),
            X = object1.X + (factor * dX),
            Y = object1.Y + (factor * dY)
        };
    });
``` 

## Possible improvements

The code above works reasonably well. The logic to interpolate 12000 missing points between over two lists runs in roughly 40ms on my Surface Book 2. A possible improvement could be to cache the previously found non-empty value in front of the interpolation target, and use that one again as long as the index of the interpolation target is less than the index of this point.
