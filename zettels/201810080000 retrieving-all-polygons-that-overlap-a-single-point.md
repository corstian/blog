---
title: "Retrieving all polygons that overlap a single point"
slug: "retrieving-all-polygons-that-overlap-a-single-point"
date: "2018-10-08"
summary: ""
references: 
  - [[202007230000 high-performance-2d-radius-search]]
  - [[201801230000 visualizing-airspace-usage-by-glider-aircraft]]
---

#software-development #dotnet

It happens more often than not when working with spatial data I have at least a reference to NetTopologySuite (NTS). NTS has a very useful implementation of an r-tree which is the `STRtree`. This object allows insertion of 2D spatial data (shapes and points), and allows querying those. My use-case is to store all kinds of polygons in the `STRtree` object and to retrieve some of them based on a single point. I want to retrieve polygons that overlap a single point.

After googling for a while I think I'm looking for something like a [Bounding Volume Hierarchy (BVH)](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy). I do think the `STRtree` is a type of a BVH. I did not dig any deeper into it, especially since I'm too lazy to write a decent data type myself, so we're stuck with an `STRtree` for now.

## Why I don't think it works as intended

The `STRtree` object has a `Query` method which accepts an `Envelope` object. This `Envelope` object also accepts a `Coordinate` object in it's constructor, and therefore only representing a single point.

When using this method I expect to be returned with all polygons that overlap the specified point. While it does not do this, it'll probably return points, if, provided at that position.

## What I can do to make it work 'as intended'

As the `STRtree` seems to do intersections, we're going to deal with this. The idea is that we draw a line from the edge of the bounding box to the point where we're at in order to retrieve all polygons that could potentially be related. I'm executing the following steps:

1. Determine what side of the bounding box is closest to the point you want to find.
2. Draw a line from the side of the box to the single point.
3. Add an additional filter on the retrieved objects to determine whether the point is actually inside any of the polygons.

For the circumstances I do think this is a fairly elegant way of retrieving this data. It's not too performant though. I only manage to do 20,000 lookups a second this way. I want it to be faster, and I do think it can be a lot faster. Turned out I am right, but more on that later.

Let's get some code going. First we need to populate the `STRtree` object. Just the same as usual.

```csharp
STRtree<object> _tree = new STRtree<object>();

// Do something like this a few times, just hook your own data.
var polygon = new Polygon();
_tree.Insert(polygon.EnvelopeInternal, polygon);

// And build the tree to start using it.
// Note that you cannot modify the contents of the tree after this.
_tree.Build();
```

Now in order to retrieve some useful data from the tree I'm using the following helper method. I highly recommend you to modify this code to your specific needs!

```csharp
public IEnumerable<object> GetPolygons(Coordinate point)
{
    // Figure out which boundary is closest.
    var xBoundary = point.X.Closest(
        _tree.Root.Bounds.MinX,
        _tree.Root.Bounds.MaxX);

    var yBoundary = point.Y.Closest(
        _tree.Root.Bounds.MinY,
        _tree.Root.Bounds.MaxY);

    var dX = point.X.Difference(xBoundary);
    var dY = point.Y.Difference(yBoundary);

    Envelope envelope;

    if (dX < dY)
        envelope = new Envelope(xBoundary, point.X, point.Y, point.Y);
    else
        envelope = new Envelope(point.X, point.X, yBoundary, point.Y);

    // Note that my object has a Polygon property which contains the polygon itself.
    return _tree.Query(envelope).Where(q => q.Polygon.Contains(point));
}
```

Note that this code only manages to execute 20,000 lookups each second. If that's good enough for you, well, go use it! If not, read on.

## Bonus: More bang for your bucks 🎉

Now if you need to squeeze a little more performance out of it I have good news for you: that's totally possible. By the usual wizardy with the Visual Studio profiler you can see the `Contains()` method takes the biggest chunk of CPU time. I'm not sure at all what takes so much processing time, but I have a gut feeling it can be a lot faster.

As I went googling a bit I stumbled upon [this answer on StackOverflow](https://stackoverflow.com/a/14998816/1720761) which suggested checking whether a point is on a single side of a polygon. I have slightly adapted the code to my use-case:

```csharp
public static bool IsPointInPolygon(Coordinate[] polygon, Coordinate testPoint)
{
    bool result = false;
    int j = polygon.Count() - 1;
    for (int i = 0; i < polygon.Count(); i++)
    {
        if (polygon[i].Y < testPoint.Y && polygon[j].Y >= testPoint.Y || polygon[j].Y < testPoint.Y && polygon[i].Y >= testPoint.Y)
        {
            if (polygon[i].X + (testPoint.Y - polygon[i].Y) / (polygon[j].Y - polygon[i].Y) * (polygon[j].X - polygon[i].X) < testPoint.X)
            {
                result = !result;
            }
        }
        j = i;
    }
    return result;
}
```

Which you can use as follows:

```csharp
_tree.Query(envelope).Where(q => Extensions.IsPointInPolygon(q.Polygon.Coordinates, point));
```

Using the code above I can manage to do about 260,000 lookups each second. While these measurements are far but scientific it shows roughly a 13x increase in performance, which is great!
