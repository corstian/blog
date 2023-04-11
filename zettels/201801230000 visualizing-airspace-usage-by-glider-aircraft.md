---
title: "Visualizing airspace usage by glider aircraft"
slug: "visualizing-airspace-usage-by-glider-aircraft"
date: "2018-01-23"
summary: ""
references: 
  - [[201810080000 retrieving-all-polygons-that-overlap-a-single-point]]
  - [[201805190000 achieving-one-billion]]
  - [[201712060000 mountain-gliding-2015]]
  - [[201705020000 cumulus-granitus-soaring-through-the-mountains]]
---

#software-development #skyhop

In the past year I have done a significant amount of work to process flight information from glider aircraft. For reference, in 2017 I have processed about 1,000,000,000 position updates and at the end of the year I had a database which stored about 700,000,000 data points, which represented over 30,000 unique aircraft. One data point contains information like latitude, longitude, altitude, speed, heading, climb-rate and turn-rate.

> *Funny thing; the difference between the amount of processed points and stored points came from beginning of the year, when I accidentally deleted the database, containing 300,000,000 data points, during a migration. Oooops. Should've made proper back-ups!*

A while ago I decided to export just a little bit of data to look into in detail. The data export contained all data from Sunday 13th of August, 2017, on which the 19th edition of the FAI European Gliding Championships has been held from Lasham Airfield.

<div dangerouslySetInnerHTML={{
  __html: `<iframe src="https://www.facebook.com/plugins/video.php?href=https%3A%2F%2Fwww.facebook.com%2Fbritishglidingteam%2Fvideos%2Fvb.742013592522273%2F1580033035386987&amp;width=500&amp;show_text=false&amp;height=280&amp;appId" width="500" height="280" style="border:none;overflow:hidden" scrolling="no" frameborder="0" allowtransparency="true" allowfullscreen="true"></iframe>`
}} />

To get an idea of the sheer size of the event, check out the video from the British Gliding Team above. In order to get an idea of the glider activity, on this day alone, check the image below.

![Glider activity over the United Kingdom on Sunday the 13th of August, 2017.](/uploads/gb_x_1625_1038x_08e65fbe3a.png)

Please note that on this day alone about 14,000,000 position update have been recorded. Considering an aircraft sends out an position update about once a second that means a little less then 4000 hours of flight activity has been recorded, on this day alone.

I managed to create a quick visualization of gliders coming back to Lasham airfield starting at around 15:30Z. See below!

<div dangerouslySetInnerHTML={{
  __html: `<iframe class="instagram-media instagram-media-rendered" id="instagram-embed-0" src="https://www.instagram.com/p/Bc8L2AhljcI/embed/captioned/?cr=1&amp;v=8&amp;wp=1316#%7B%22ci%22%3A0%2C%22os%22%3A160.85500000000002%7D" allowtransparency="true" frameborder="0" height="776" data-instgrm-payload-id="instagram-media-payload-0" scrolling="no" style="background: rgb(255, 255, 255); border: 1px solid rgb(219, 219, 219); margin: 1px 1px 12px; max-width: 658px; width: calc(100% - 2px); border-radius: 4px; box-shadow: none; display: block; padding: 0px;"></iframe>`
}} />

<br />The things I'm displaying here represent only a small portion of the work which has been done, and an even smaller portion of the full potential of this data in combination with the right tooling. Although this by far isn't the end result I'm wanting to achieve I'm already quite proud on this humble result! More and better stuff will definitely follow in the future! Stay tuned for further updates! :)
