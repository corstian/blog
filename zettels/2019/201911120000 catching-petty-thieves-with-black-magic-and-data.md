---
title: "Catching petty thieves with black magic and data"
slug: "catching-petty-thieves-with-black-magic-and-data"
date: "2019-11-12"
summary: ""
references: 
---

#software-development

> This post sat in my drafts for over a year now. The contents yield no result, but I hope this might inspire you to do something crazy with your day. Honestly, [this idiot whom ran over a cyclist](https://twitter.com/Thund3r_H4wk/status/1194073978123493377?s=20) inspired me to publish this post.
> <blockquote class="twitter-tweet"><p lang="en" dir="ltr">Last week&#39;s video highlighted the dangerous design of NYC bike lanes. Today a cyclist in a bike lane (yes, that&#39;s a bike lane) sandwiched between two driving lanes was rear-ended by a van. <a href="https://twitter.com/hashtag/VisionZero?src=hash&amp;ref_src=twsrc%5Etfw">#VisionZero</a> appropriately describes the street lighting. <a href="https://twitter.com/StreetsblogNYC?ref_src=twsrc%5Etfw">@StreetsblogNYC</a> <a href="https://twitter.com/TransAlt?ref_src=twsrc%5Etfw">@TransAlt</a> <a href="https://twitter.com/NYC_DOT?ref_src=twsrc%5Etfw">@NYC_DOT</a> <a href="https://t.co/L1XvWf7YY6">pic.twitter.com/L1XvWf7YY6</a></p>&mdash; Jessica (@Thund3r_H4wk) <a href="https://twitter.com/Thund3r_H4wk/status/1194073978123493377?ref_src=twsrc%5Etfw">November 12, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


This post is just for shits and giggles. While the actual goal is to catch some petty thieves which did about €400 in damages to a van, I'm not exactly sure whether I'll succeed in that at the time of writing. Let me tell you a little bit of the backstory first.

So this all started with some petty thieves driving along our company's building and seeing a lone van standing in the parking lot. Not really thinking brightly early in the day they decided to look what's in it. Not looking up or around they approach the van, look into it, and decide it's worth breaking it open for.

Best thing? It's all caught on camera.

<div dangerouslySetInnerHTML={{
  __html: `<iframe src="https://www.facebook.com/plugins/video.php?href=https%3A%2F%2Fwww.facebook.com%2F2013159872267013%2Fvideos%2F311783256263732%2F%3Fhc_ref%3DARRvcoM09jKIzwgGLT9Jhjn_Hyo5iGBDiLZv-lwo-MavGdLULqkZxMeOpJRWidg4S8A%26fref%3Dnf%26__xts__[0]%3D68.ARCH2oCy7Lgcw0DqYqC6vOxXCQ8kWh-cABAne4KnbnffzAirvevWWSzY-suYjATQND7cG2jgC0Rw0tt0T3OgrbKYrlaLxy0Z6tAcHP-DcfuTPNbDSihF6KhLHZU3zMsNgvtU2yqx6K6c2ybOiu30mSphIjC-H9nNR2SRE5gM8GmJP5EhZbWp%26__tn__%3DkC-R&amp;width=500&amp;show_text=false&amp;height=280&amp;appId" style="border:none;overflow:hidden" scrolling="no" allowtransparency="true" allow="encrypted-media" allowfullscreen="true" class="present-before-paste present-before-paste" width="500" height="280" frameborder="0"></iframe>`
}} />

What's our problem then? Didn't we get 'em? Well, not exactly. We're not able to see their license plate due to overexposure. First thing we'd be trying was to alter the exposure of our imagery to try to get to see some numbers, but no. That did not work.

## Digging down

So what are our options here? What information do we have, and how can we get some more information?

**The car:** the car seems to be a Citro&euml;n C4 manufactured somewhere between 2005 and 2010. Seems to have a somewhat light color.

![http://localhost:3000/uploads/vlcsnap-2018-09-17-13h54m35s658.jpg](/uploads/vlcsnap_2018_09_17_13h54m35s658_7c9bed6b6a.jpg)

**License plate:** Not known

And this is what is known. Our ultimate goal is to get to know the license plate. So we have to combine all information we have from the video feeds, and possibly more. The most information we can get about the license plate is this still as they're driving away.

![http://localhost:3000/uploads/drivingaway.png](/uploads/drivingaway_3f68367975.png)

We have pixels! Playing around with our color curves we can get the pixels more clear! (Actually I'll use the inverse of the picture, making it easier working with a white background.)

![http://localhost:3000/uploads/curvy.jpg](/uploads/curvy_46be4f4a63.jpg)

Next step, we're going to check if we can get pixels to match up with the license plate! How? We get the same font, and we're going to make sure it's about 3 pixels high. Yup. Should work. We could brute-force the license plate, but why bother? There's only a limited set of possibilities. See [this Wikipedia page](https://en.wikipedia.org/wiki/Vehicle_registration_plates_of_the_Netherlands) for more information about the license plate system in use in the Netherlands. But first let's narrow down the possible license plates. In order to be able to do this we will need to know when this car was manufactured. Based on the look of the brake-lights I'd say it was somewhere between 2010 and 2015. (For reference pictures; visit https://www.cars-data.com/en/citroen-c4-2010/445)

According to Wikipedia the following ranges of license plates have been issued to cars:

* 00-**K**BB-1, registration 2009/2010
* 00-**L**BB-1, registration 2010
* 00-**N**BB-1, registration 2010/2011
* 00-**P**BB-1, registration 2011
* 00-**R**BB-1, registration 2011
* 00-**S**BB-1, registration 2011
* 00-**T**BB-1, registration 2012
* 00-**X**BB-1, registration 2012
* 00-**Z**BB-1, registration 2012/2013
* 1-**K**BB-00, registration 2013
* 1-**S**BB-00, registration 2013
* 1-**T**BB-00, registration 2013/2014
* 1-**X**BB-00, registration 2014
* 1-**Z**BB-00, registration 2014/2015
* **G**B-001-B, registration 2015
* **H**B-001-B, registration 2015/2016

To limit the possibilities even more, registrations with 'SD' or 'SS' are not issued. According to Wikipedia: "Nowadays the letters used do not include vowels, so as to avoid profane or obscene language. To avoid confusion with a zero, the letters C and Q are also omitted. Letters and numbers are issued in strict alphabetical/numeric order.". This effectively leaves us with **B**, **D**, **F**, **G**, **H**, **J**, **K**, **L**, **M**, **N**, **P**, **Q**, **R**, **S**, **T**, **V**, **W**, **X** and **Z** (19 characters) to use as letters for use in license plates. But why bother calculating all these possible combinations? There's a data set we can use!

## Datasets! 🎉

So the RDW (Dutch instance responsible for issuing license plates) has some datasets which are freely available (see https://opendata.rdw.nl/ for the sets). One of these contains the license plates in combination with quite a lot of information about the car. Information like the brand, make, manufacture date and more technical information than I care to know are in this dataset. After downloading all this data as a CSV file it ended up being a 7.1Gb extraction. Nice.

We're going to filter this data set with the following criteria:

* **Brand:** Citro&euml;n
* **Make:** C4, 5-door hatchback
* **Type:** Passenger car
* **Registration:** Something which is inside of the criteria outlined above.
* Still allowed on the road
* Accepted on the road somewhere between 2010 and 2015 (basically an alternative filter for the registrations)

![http://localhost:3000/uploads/screenshot.png](/uploads/screenshot_602e37b23b.png)

After the first filter session we're down to 4987 possible cars. Nice. Compared with the 14.1 million records in the dataset we're only working with about 0.12% of the original amount of data. Looks good.

Let's see if there are ways to generate images which have a quality which is at least as bad as our security cameras. A quick google search came up with [this StackOverflow thread](https://stackoverflow.com/a/2070493/1720761). As we're doing nothing fancy here we can just copy paste this into our LinqPad window. Just a quick test to see whether it works and voila. I think this is something we can continue with.

![Generating images containing letters of license plates.](/uploads/anotherscreenshot_c26aa44f25.png)
*Code generating images containing letters representing license plates.*

But our problem is that the result is so bad we can barely make it up. Okay. Resize it! [Back to StackOverflow](https://stackoverflow.com/a/24199315/1720761). The resulting image looks like this. (Browser resizing makes the image blurry. I'm too lazy to turn that off right now)

![License plate, but resized to a nearly unreadable format.](/uploads/img_637d023b18.bmp)

Just about right if you'd ask me. As we only have a 3x13 grid of pixels.

## Proceeding further?

Now what? We'd have to match bit arrays. Given this data is pretty high dimensional I have decided to use K-Means clustering to try and find the relative distance between number plates, and then select the platest which were most closely related. Even though the proof of concept worked, and I could produce results, I don't really believe this method is practically viable. Different factors like optics, skewed angles and a far from accurate depiction of a number plate are all problems which have to be experimented with and validated before they can be used in a project like this.

All these factors have made it so that I lost interest in pursuing this experiment any further. While it 'might' work, there are many unknowns which makes this experiment even more complex than it already seemed to be initially.

For those who are actually interested in the technical part, I have posted the script I used for all this [on GitHub](https://gist.github.com/corstian/51f4a709c80f4f463b22eea4d3217dd0)!

