---
title: "Representing coordinates in a human readable way"
slug: "representing-coordinates-in-a-human-readable-way"
date: "2019-02-25"
summary: "In an attempt to make coordinates legible without resorting to maps I have tried an approach where I describe the position relative to a know point, such as a city, mountain top, or other landmark."
---

#software-development #dotnet

> **Download the source-code here!**  
> *In order to get you up and running quickly you can find [the gist of the code here](https://gist.github.com/corstian/8ac817cc378c56de69b43aff8cf398f2#file-coordinatestotext-linq) or [download a zip containing the LinqPad snippet here](https://gist.github.com/corstian/8ac817cc378c56de69b43aff8cf398f2/archive/aa203b43f7c4ef2d9438761d31b7e993a540d6e3.zip).*

## <a id="contents">0.</a> Contents

1. [Introduction](#introduction)
2. [The plan](#plan)
3. [The code](#code)  
  3.1 [Resolving landmarks](#landmarks)  
  3.2 [Mathematical background](#math)  
  3.3 [Calculating the distance between points](#distance)  
  3.4 [Calculating the angle between points](#angle)  
  3.5 [The result](#result)

## <a id="introduction">1.</a> Introduction

A single coordinate is just a set of numbers indicating a location in a grid of a certain dimension. These numbers themselves are intuitive for humans to interpret at all. We're going to explore how to turn these numbers into a textual representation which can easily be understood by the common people. While the precision of the coordinate system is higher and more exact, I for myself would be happy if I can tell the difference between Amsterdam, Paris and Berlin based on a set of coordinates.

> [For more information about the accuracy each decimal in a coordinate represents, see this answer on the GIS StackExchange.](https://gis.stackexchange.com/a/8674)

We're going to try something different. As we can say human interpretation of coordinates is shit, why not try to represent coordinates as something which is nuts for a computer to work with, but actually useful for humans to interpret.

We are going to translate a coordinates into a textual representation which describes the position relative to a known landmark (e.g. a city or mountain). This has been inspired by the way positions are communicated over the radio in aviation.

> Aside; if you want to get to know or refresh your knowledge about radio technique in aviation, [check out this guide, appropriately called "VFR COMMUNICATIONS FOR IDIOTS"](https://www.westwingsinc.com/vfrcomm.pdf).

## <a id="plan">2.</a> The plan

My primary use case for this technique will be aviation related. Aircraft are moving all the time, and considering separation of gliders happens visually (meaning the pilots just have to look outside) an accuracy of about 2km is good enough for me.

As we will be using local reference points our result shall be something in the likes of "2 miles north of Steenbergen at 1200 feet". This little line contains all the information we need to accurately identify a location:

1. A local (well known) reference point.
2. A direction from the reference point (360 degrees).
3. A certain distance from the reference point following the heading.
4. An altitude. Always easy in aviation.

We could always add more information about the movements of the aircraft (like heading north-northwest at 50 knots), but that should be trivial in relation to this post.

## <a id="code">3.</a> The code

We're going to write a little library which will solve most of the problems of translating coordinates to a human readable format. [A while ago I made a pull-request to the Humanizer repository which added degree-heading to word capabilities](https://github.com/Humanizr/Humanizer#heading-to-words), which will help us along the way later.

### <a id="landmarks">3.1</a> Resolving landmarks

First we have to retrieve data about local landmarks. Essentially what we need is a dictionary (list of key-value pairs) which contain coordinates and a label. Thankfully there are sources on the internet which provide databases of (big) cities and their coordinates we will use for this post. One of these is the [GeoNames project](https://www.geonames.org/). You can download various types of files for free [from here](http://download.geonames.org/export/dump/). The file I go with is the \`[cities5000.zip](http://download.geonames.org/export/dump/cities5000.zip)\` file in order to get a text file which contains all cities with more than 5000 inhabitants.

There are various ways to efficiently filter through a dictionary with location data. For starters I would recommend putting this information in a database and to use the database engine to do spatial queries. I am not going to cover this, and for the sake of the demo I'll just use a \`Dictionary\` to store and filter the data in-memory.

We will need a helper method to filter out the cities closest to a given location. Since NTS (NetTopologySuite) is super useful in general when dealing with spatial data we're using it's [K-D tree](https://en.wikipedia.org/wiki/K-d_tree) implementation for filtering through location data.

Using it is not too difficult. Honestly, in a few lines we could populate it and find the nearest position to a point. Don't mind my helper methods. You'll get the gist of it.

```csharp
var tree = new KdTree<LocationEntry>();

GeoNames.ExtractData(file)
    .ToList()
    .ForEach((i) => tree.Insert(new Coordinate(i.Latitude, i.Longitude), i));

var landmark = tree.NearestNeighbor(coordinate);
```

### <a id="math">3.2</a> Mathematical background

We need to know the angle from the point of reference to the coordinate we want to make more human readable. The math to do this has been figured out by people way smarter than me and has been around for a long time. [There is this website that does a great job explaining latitudal/longitudal calculations, and I recommend you check it out if you want to know all about it](https://www.movable-type.co.uk/scripts/latlong.html). It features some interactive calculations and shows the math right there with some calculation.

### <a id="distance">3.3</a> Calculating the distance between points

In order to calculate the distance we use the haversine formula to calculate the shortest route over a sphere between two points, also known as the short circle distance.

The following code is derived from the `GeoCoordinate` class in the `System.Device.Location` namespace (`System.Device.dll` assembly). [A .NET Standard port can be found here](https://github.com/ghuntley/geocoordinate/blob/master/src/GeoCoordinatePortable/GeoCoordinate.cs).

```csharp
public static double DistanceTo(double lat1, double long1, double lat2, double long2)
{
    if (double.IsNaN(lat1) || double.IsNaN(long1) || double.IsNaN(lat2) ||
        double.IsNaN(long2))
    {
        throw new ArgumentException("Argument latitude or longitude is not a number");
    }

    var d1 = lat1 * (Math.PI / 180.0);
    var num1 = long1 * (Math.PI / 180.0);
    var d2 = lat2 * (Math.PI / 180.0);
    var num2 = long2 * (Math.PI / 180.0) - num1;
    var d3 = Math.Pow(Math.Sin((d2 - d1) / 2.0), 2.0) +
                Math.Cos(d1) * Math.Cos(d2) * Math.Pow(Math.Sin(num2 / 2.0), 2.0);

    return 6376500.0 * (2.0 * Math.Atan2(Math.Sqrt(d3), Math.Sqrt(1.0 - d3)));
}
```

### <a id="angle">3.4</a> Calculating the angle between points

In order to calculate the angle between two points we use a "rhumb line". The rhumb line is a line you can follow from one point to another by following the same compass heading. Note that this is not the short circle distance, which we talked about earlier. For most applications the distances are so small that it doesn't really matter anyway.

The following code to calculate the angle of the rhumb line is copied from [this StackOverflow answer](https://stackoverflow.com/a/2042883/1720761).

```csharp
public static double DegreeBearing(
        double lat1, double lon1,
        double lat2, double lon2)
{
    var dLon = ToRad(lon2 - lon1);
    var dPhi = Math.Log(
        Math.Tan(ToRad(lat2) / 2 + Math.PI / 4) / Math.Tan(ToRad(lat1) / 2 + Math.PI / 4));
    if (Math.Abs(dLon) > Math.PI)
        dLon = dLon > 0 ? -(2 * Math.PI - dLon) : (2 * Math.PI + dLon);
    return ToBearing(Math.Atan2(dLon, dPhi));
}

public static double ToRad(this double degrees)
{
    return degrees * (Math.PI / 180);
}

public static double ToDegrees(this double radians)
{
    return radians * 180 / Math.PI;
}

public static double ToBearing(this double radians)
{
    // convert radians to degrees (as bearing: 0...360)
    return (ToDegrees(radians) + 360) % 360;
}
```

It might be interesting to you to figure out the difference between "heading", "bearing" and "course" if you do not already know. [Someone described the differences here](https://diydrones.com/profiles/blogs/the-difference-between-heading).

### <a id="result">3.5</a> The result

I bet you could come up with this yourself, but essentially we have all the individual components to for a textual representation of a coordinate.

We figured out:

* The position we need to resolve
* The closest landmark
* The distance to the landmark
* The heading to the landmark

It's fairly simple to put it together now:

```csharp
var text = $"{distance}km {bearing.ToHeading(HeadingStyle.Full)} of {landmark.Data.Name}";
```

Which might result in "3km south of Bergen op Zoom".

