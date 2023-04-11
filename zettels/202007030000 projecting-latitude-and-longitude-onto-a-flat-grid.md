
---
title: "Projecting latitude and longitude onto a flat grid"
slug: "projecting-latitude-and-longitude-onto-a-flat-grid"
date: "2020-07-03"
summary: ""
---

#software-development

Sometimes it's just easier to deal with a flat grid representing coordinates than to deal with certain cartesian projections. Though I understand that these projections are there for a reason, there are situations where it might be beneficial to have a simple grid representing the number of kilometers from the equator as Y value and the number of kilometers from the Greenwich meridian as X value. Read on for some quick and dirty map projection magic;


Based on [this answer on the GIS StackExchange](https://gis.stackexchange.com/a/142327/129511) I figured I could easily use the math used to calculate an estimation of the longitudal distance to 0 (Greenwich mean line) in kilometers.

```csharp
public struct Point
{
	public double X;
	public double Y;
}


public static Point FromCoordinate(double latitude, double longitude)
{
    return new Point
    {
        X = Math.Cos((Math.PI / 180) * latitude) * 111 * longitude,
        Y = latitude * 111
    };
}
```

To be aware of the subtle inaccuracies when using this code I think it is important to understand how the latitudal and longitudal coordinate system works (also mentioned in the answer over at GIS StackExchange):

> Degrees of latitude are parallel so the distance between each degree remains almost constant but since degrees of longitude are farthest apart at the equator and converge at the poles, their distance varies greatly.
> 
> Each degree of latitude is approximately 69 miles (111 kilometers) apart. The range varies (due to the earth's slightly ellipsoid shape) from 68.703 miles (110.567 km) at the equator to 69.407 (111.699 km) at the poles. This is convenient because each minute (1/60th of a degree) is approximately one [nautical] mile.
> 
> A degree of longitude is widest at the equator at 69.172 miles (111.321) and gradually shrinks to zero at the poles. At 40° north or south the distance between a degree of longitude is 53 miles (85 km)


Using my location as a reference point there's roughly a .5% error using this approach when measuring the distance to the Greenwich line. Something which is, depending on the use-case, quite acceptable.

