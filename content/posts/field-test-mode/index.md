---
title: "Field Test Mode"
date: 2024-11-18T17:37:08-04:00
---

I've been having a lot of call trouble at home lately using [Eastlink](https://eastlink.ca/) and an iPhone 13 mini.

Even though I have all the signal bars:

{{< figure
  alt="A screenshot of the top of my phone's home screen showing full cell signal bars."
  caption="Full bars"
  src="bars.png" >}}

Received and placed calls would fail right away or audio would cut in and out for both sides.

I've been working with Eastlink support but no breakthroughs yet.

Today, I got to thinking there must be a way to see more detailed signal/noise info.

That led me to learn about Field Test Mode!

If you dial `*3001#12345#*` no call is placed but you get a dashboard with this and other info:

{{< figure
  alt="A screenshot of the Field Test Mode dashboard showing various signal values."
  caption="Initial look at Field Test Mode"
  src="bad.png" >}}

Using [this site](https://www.waveform.com/a/b/guides/field-test-guide#field-test-mode-for-iphones) for reference,
I think it makes sense now why I have all the signal bars but my calls are poor.

My RSRP (signal strength) value is pretty good but my SINR (interference and noise) values are bad.

Going into Settings > Cellular > Network Selection, I tried forcing my network to one of the roaming networks that appear if I turn off Automatic.

Looking at the Field Test Mode dashboard after that, I see:

{{< figure
  alt="A screenshot of the Field Test Mode dashboard showing various signal values. Some values are improved from the previous screenshot."
  caption="Better?"
  src="better.png" >}}

The stregth value is about the same but the interference/noise values are much better.
Trying a test call to Eastlink's support number works and doesn't cut out!

This may not be sustainable but it's nice to have an option for now.
