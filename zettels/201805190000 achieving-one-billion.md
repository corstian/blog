---
title: "Achieving one billion"
slug: "achieving-one-billion"
date: "2018-05-19"
summary: ""
references: 
  - [[201801230000 visualizing-airspace-usage-by-glider-aircraft]]
---

#software-development #skyhop #data-storage

Over the last years I've been working continuously with large data sets. Whether it's about air quality, aircraft movements, plant growth or weather information. I love to process 'pretty big data' as I call it. Putting this data into data stores, processing it as fast as possible, preferably real-time or near real-time, and building cool applications on top of this data.

Yesterday is the day that one of my projects passed the 'magic' 1 billion record count. Lets look at all the stuff that got out of hand so badly that this could've happen.

## Fiddling with software

A really long time ago when I was just a little kiddo, possibly somewhere about 14 years old, I was playing with some PHP websites and scripts. Figuring out how things were working, trying to write some new code, and trying to get the computer to do what I was trying to achieve. It was at this time that I was trying to write a contact form which should've send the input in the form as email to my own mailbox. I'm not sure what I did wrong but in one way or another the script to send the mail got stuck in an infinite loop and started continuously sending mail to my own email address, which was hosted at my father's company.

Five minutes later my father called me. They got a call from the ISP that they (me) were taking down their server(s?) with a mail bomb. These emails caused carnage everywhere in between my little web server and the receiving mail server (at my father's company). In the timespan of only five minutes I ended up overloading a few mailservers (which were probably incorrectly configured either way), and about 80,000 emails in my own mailbox.

That's only 25 mails per second.

This was the last moment ever I underestimated the power of computers.

## Learning about databases

Around the same time that I set off the email bomb I also learned about databases. Well, I knew there were tables, and I knew I could store and retrieve data with them, but don't you dare ask me about foreign keys, indices and other technical details. What did I know? When a classmate showed me a database which contained a table with about 40,000 records I was amazed. How could one get to fill a database with so much information? I could not imagine I was ever about to create a database so big.

A while later I met some people who wanted to build an online platform which would help consumers find the perfect car for their needs based on a set of questions. For this system to work we needed a bit of data about different cars so I wrote a tool to scrape some websites and put this information in a database. After letting this scraper run for a few nights we ended up with information about 400,000 different types of cars. Note I still did not know about foreign keys and all that stuff, but the numbers were getting bigger.

## Growing bigger

When I was 17 years old I started writing software at my father's company. It was this time that I wrote the base for a system which now processes information from about 1000 IoT like devices in real-time. It was at this time that I started learning about database internals because I had to. I needed it to keep the response times in an acceptable window.

I remember searching around on google about experiences people had with large data sets in order to be able to estimate the amount of resources needed. I was worried the whole thing would burn down as soon as 13 million records were processed. These numbers felt so big that I could not possibly imagine how much resources were needed to work with it. And amazingly it continued to work to this day, with 16 million records and counting.

## … and bigger

These days my biggest side project is about processing flight information from glider aircraft. When I started one and a half year ago I had no idea about how fast this project was about to grow. In less then a year the tool processed one billion data points in real-time, sometimes peaking at about 20,000 points / second, amounting to millions of points each day.

And then there are the learning moments in this traject. Like trashing a live database with 300 million records, and not having backups. The moments I've been trying to debug and fix the data processing beast without causing any downtime, and all moments I was busy doing performance optimization and I did not think I was going to make it. It's funny how both the idea and the program evoluted at the same time. In the beginning I just wanted to process a small amount of data, and it took 10 seconds to process 7,000 records. Right now the same amount of information can be processed and distributed in just shy of a quarter of a second.

And this way the project starting to grow, and continued growing, regardless of the moments I just wanted to trash it because I did something stupid, or the moments I did not have any energy to continue working on it, or because of any other reason there was to stop.

And now it contains 1,000,000,000 data points. Actually I'm more proud about this achievement then I care to admit. It feels like an amazing milestone for me personally 🎉.

A quick calculation shows that these one billion data points represent more then 60 full years of experience flying aircraft in all kinds of conditions. This is more then 20,000 full days.

<div dangerouslySetInnerHTML={{
  __html: `<iframe class="instagram-media instagram-media-rendered" id="instagram-embed-0" src="https://www.instagram.com/p/Bc8L2AhljcI/embed/captioned/?cr=1&amp;v=8&amp;wp=1316&amp;rd=app.cloudcannon.com#%7B%22ci%22%3A0%2C%22os%22%3A274%7D" allowtransparency="true" frameborder="0" height="870" data-instgrm-payload-id="instagram-media-payload-0" scrolling="no" style="max-width: 658px; width: calc(100% - 2px); background-color: white; border-top-left-radius: 3px; border-top-right-radius: 3px; border-bottom-right-radius: 3px; border-bottom-left-radius: 3px; border: 1px solid rgb(219, 219, 219); box-shadow: none; display: block; margin: 0px 0px 12px; padding: 0px;"></iframe>`
}} />

<script async="" defer="" src="//www.instagram.com/embed.js"></script>

## What's next?

Data alone is not useful. There is currently so much data stored that no one will understand the context of the data on itself without proper aggregation. One of the biggest goals for the future is to streamline the data aggregation and information distribution processes.

My goals are the following:

* Develop or set up an information processing pipeline which allows for rapid development of new data processing steps
* Develop API's so that this information can be shared with the world
* Apply machine learning to this data set in order to be able to predict several things, including thermalling hotspots based on the weather conditions
* Develop a proper user interface and attract pilots to use this platform for their flight analysis needs

The goals above will probably take several years to achieve. Nevertheless, on to the next 10 billion data points and amazing data processing tools!
