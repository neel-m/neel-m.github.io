---
layout: post
title: "My First Go Module"
author: "Neel Malik"
date: 2023-05-27
tags: GO
---

Created my first Go Module today: [replacements](https://pkg.go.dev/github.com/neel-m/replacements)

This all started because I wanted to learn Go.  The only way to truly know something is to use it for something "real".  Then to prove that you really understand it, explain it someone else who doesn't already know it.

It is dehumidifier season again, so the dehumidifier in my basement is back to coming on again.  Some time ago, I hooked it up to a control system.  Turns out it was so long ago that I did this, that I didn't remember exactly what was controlling it.  After some digging through all the various iterations of system I still have running, I discovered it was OpenHAB running on a very old Surface Laptop.  That is the n-2 system that has been running for several years and it is on the list to decommission.

It was using a very simple control algorithm:
* if the 60 min average relative humidity at the input to the dehumidifier is greater than the setpoint (60%) turn on the dehumidifier
* if the 60 min average is less than setpoint - 5, turn off the dehumidifier

Based on the graphs that I had (Grafana and InfluxDB) this seemed to work well.  I am not using OpenHAB on the new system.  I am sure it has improved in the years it has been since I updated it last, but I never found it to be easy.

I created the new control using NodeRED, but changed the setpoint to 55%.  The first few iterations looked fine, but there were some anomalies where it appeared to be short cycling.  This is another story, but the Flux query for average wasn't doing what I wanted.  So, I figured out a different query that works much better.

With the control system working better, I wanted to get a better idea of the efficiency.  I have a modified rain gauge that measures the water that the dehumidifier outputs.  I "calibrated" the gauge and it counts 200 buckets per cup.  Looking at the graph of bucket counts (and power) over time, it is clear that it takes the dehumidifier several mins (after the compressor starts) to start condensing water.  It also continues putting out water for some time after the compressor stops.

I did come up with a complex Flux query that gives me a good indication of the efficiency.  Since Flux typically works on a single row it was a bit challenging to do this.  There are three measurements to get the data:
* bucketCounter (monotonically increasing counter of the number of buckets of water from the rain gauge)
* powerControl (the dehumidifier is on or off)
* totalPower (monotonically increasing counter of kWh used by the dehumidier)

With 2 full joins and several filter(), difference(), keep(), map() I was able to get a query that worked.  It worked fine for a day or so.  But, when I asked it to do several days, it took more than 30s to return.  Since the sensors are all reporting data every 30s, this is not too surprising.  The other problem with this query is that it counts all the buckets from the last time the dehumidifier shut down to the current time it shut down.  This means it is counting buckets that really belong to the previous cycle.  It also means that it will count possibly false readings from when the dehumidifier is off for a long period.  I wanted to count the buckets from just before the dehumidifier starts to about 5 min after it stops.

InfluxDB has an API. How hard could it be to write something to do a more targeted calculation.  Since I have been using Python for awhile, I started there.  This is the query I use to get the on/off times for the dehumidifier:

```Flux
from(bucket: _bucket)
  |> range(start: _start, stop: _stop)
  |> filter(fn: (r) => r["NodeName"] == "dehumid")
  |> filter(fn: (r) => r["_measurement"] == "PowerControl")
  |> filter(fn: (r) => r["_field"] == "POWER")
  |> difference()
  |> filter(fn: (r) => r["_value"] != 0)
  |> elapsed()
```

difference() changes the POWER field to make it easy to filter out the entries where there is no change.  elapsed() makes it easy to get the time between entries, so I know the start and stop times.

The Python version of the API makes it easy to use parameters:
```params = {"_bucket": bucket, "_start": start, "_stop": now}```

I then decided to try this with Go.  The documentation for the Go version is less good.  It does seem to indicate that parameterized queries are only supported on the cloud version (not the OSS version I am using).  I couldn't get the Go version to work and the errors it was giving me led me to believe that it really didn't work with parameters.

Not willing to give up, I started debugging to call in Python to see if perhaps the Python version was doing the parameterization in Python.  Tracing through the calls, it was clear it was adding OpenAPI data for the parameters that it was then sending to the server.

I didn't find any helpful information on how to make the Go version use parameters, or really any examples of usage beyond the very limited ones in the documentation (which are less than illuminating on how to actually use it in a real program).

Since a Flux query is really just a long string, I decided the "old school" approach would likely work.  I searched the go package directory for something.  I said, "surely someone else has had this problem before me, and is way more motivated to share that solution".  Unfortunately for me, that statement turned out to be false.  But, luckily for everyone else who says that, it will now be true.

