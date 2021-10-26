---
layout: post
title:  "Converting Between GPS Time Standards"
date:   2021-10-19 21:18:32 -0400
categories: spatial
---
![](/img/als-snippet-gpstimecolored.png)

When working with airborne lidar point cloud data, there are several time standards that one may encounter:
1. *GPS Time:* Number of seconds since 6 January 1980 00:00:00.
2. *GPS Standard Time:* Number of seconds since 6 January 1980 00:00:00, less 1 billion.
3. *GPS Week Seconds:* Number of seconds since the start of a new week. Resets every week at midnight between Saturday and Sunday.

Airborne lidar point clouds stored in [LAS](https://www.asprs.org/wp-content/uploads/2010/12/LAS_1_4_r13.pdf){:target="_blank"} format almost always include timestamps, which are typically given in GPS Standard Time. However, it is not uncommon for the aircraft trajectory (path the aircraft/sensor flew whilst collecting lidar measurement) to be stored with GPS Week Seconds as the time standard. And you may occasionally encounter a lidar point cloud with timestamps in GPS Week Seconds or GPS Time rather than GPS Standard Time.

These discrepancies are usually not an issue, but when they are, there aren't many (any?) options outside of commercial software (or your own custom script) that will take care of the conversion. I occasionally run into these time differences when interpolating trajectory information (x, y, z, roll, pitch, heading) for lidar point data, which requires the trajectory and lidar point cloud timestamps to be referenced to the same time standard. After the third or fourth time dealing with this, I decided to create a [`gpstimeconvert`](https://pdal.io/stages/filters.gpstimeconvert.html) filter for the [Point Data Abstraction Library](https://pdal.io/), better known by its acronym - PDAL.

## Relevant Time Trivia

#### The GMT Time Zone
GPS time is pegged to the [GMT](https://en.wikipedia.org/wiki/Greenwich_Mean_Time) time zone. This is relevant when converting from GPS Week Seconds to GPS Time or GPS Standard Time. These conversions require you to specify when your data collection started because GPS Week Seconds are ambiguous - they repeat from 0 to 604800 (60\*60\*24\*7) every week. The most common method for specifying when your data collection started is to supply the GPS Week Number, where GPS Week Number is the number of weeks since 6 January 1980 00:00:00. You can find the GPS Week Number for a particular date via online calendars, e.g., [here](http://navigationservices.agi.com/GNSSWeb/).

But - and this is where the GMT Time Zone becomes relevant - you need to be careful if your data collection started on a Saturday or Sunday! You should convert the collection start date from your local time zone to the GMT time zone to determine if it falls on GMT Saturday or GMT Sunday. Note that you'll need a good idea of the local start time, not just the local date. If you get it wrong, you will end up with an incorrect (off by one) GPS Week Number. Of course, you can always just check the first timestamp in your data - if it is a low number, then you know your start date was a GMT Sunday. If it is close to 604800, then the start date was on a GMT Saturday.

#### Week Second Wrapping
What happens when your timestamps are in GPS Week Seconds and they span the beginning of a new week? There are two possibilities, depending on the software that wrote the timestamps:
1. *Wrapped Week Seconds:* The week seconds reset to zero at the crossover. Note that if this is the case, sorting your point data by time probably isn't going to accomplish what you intended.
2. *Unwrapped Week Seconds:* The week seconds are allowed to increase beyond 604800.

If you are converting from GPS Week Seconds and your data spans a new week, you will need to know which option was used for your data. Likewise, if you are converting to GPS Week Seconds, you will need to decide which option to apply.

#### Leap Seconds
GPS time ignores [leap seconds](https://en.wikipedia.org/wiki/Leap_second). As of this writing, 18 leap seconds have occurred since the GPS zero time of 6 January 1980 00:00:00. Therefore, GPS time is currently 18 seconds ahead of [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time) time (we live our daily lives according to UTC time). Note that this means a new GPS week starts 18 seconds before a (normal to us) UTC week. I mention leap seconds here because the (current) 18 second difference can cause confusion when comparing GPS and UTC times.

## Using the Filter

This explanation assumes you are familiar with PDAL. If not, head to their [quickstart](https://pdal.io/quickstart.html) page. I recommend a [Miniconda](https://docs.conda.io/en/latest/miniconda.html) install to get up and running with minimal pain. 

The `gpstimeconvert` filter enables conversion between the three time standards by specifying one of the following conversion types:
1. `ws2gt`: GPS Week Seconds to GPS Time
2. `ws2gst`: GPS Week Seconds to GPS Standard Time
3. `gt2ws`: GPS Time to GPS Week Seconds
4. `gst2ws`: GPS Standard Time to GPS Week Seconds
5. `gt2gst`: GPS Time to GPS Standard Time
6. `gst2gt`: GPS Standard Time to GPS Time

For example, the (trivial) `gt2gst` and `gst2gt` conversions either subtract or add a billion seconds to the timestamps. Simply specify the `conversion` type in the PDAL pipeline:

{% highlight json %}
[
    "input.las",
    {
        "type": "filters.gpstimeconvert",
        "conversion": "gt2gst"
    },
    "output.las"
]
{% endhighlight %}

Conversions from GPS Time or GPS Standard Time to GPS Week Seconds (`gt2ws` and `gst2ws` conversion types, respectively) assume the data is ordered by ascending collection time. You can force this assumption to be true by applying PDAL's `filters.sort` to the `GpsTime` dimension in a prior stage. These conversions accept a `wrap` option to specify whether to reset the output GPS Week Seconds to 0 if a new week is encountered. If the option is not specified, it defaults to `false`, which allows the week seconds to extend beyond 604800 if a new week is spanned. Here's a sample pipeline:

{% highlight json %}
[
    "input.las",
    {
        "type": "filters.sort",
        "dimension": "GpsTime",
        "order": "ASC"
    },
    {
        "type": "filters.gpstimeconvert",
        "conversion": "gst2ws",
        "wrap": "True"
    },
    "output.las"
]
{% endhighlight %}

Last, we look at converting from GPS Week Seconds to GPS Time or GPS Standard Time (`ws2gt` and `ws2gst`). As before, these conversions assume the data is ordered by ascending collection time. However, be cautious if applying a sort filter to times in GPS Week Seconds. If the times span a new week and the week seconds are not unwrapped, sorting by time will not order the data by collection time.

Because week seconds are ambiguous (the times repeat every week), a `start_date` option, in YYYY-MM-DD format, is *required* for these conversions. The start date is the GMT date on which the data collection commenced. If the collection started very close to a new week (i.e., a Saturday or Sunday), it is worth inspecting the first time in the data: a time near 604800 indicates the data was collected on Saturday, whereas a time close to 0 indicates the collection started on Sunday. As discussed earlier, a GPS Week Number is often used to address the ambiguity in GPS week seconds. However, GPS Week Numbers are not intuitive and are ambiguous given that they cycle from 0 to 1024 unless unwrapped. Hence, we use a date (in the GMT time zone!) instead.

The conversions accept a `wrapped` option to specify whether the input GPS Week Second times extend beyond 604800 or reset to 0 if the data spans the start of new week. A value of `True` indicates the input times reset to 0 for new weeks; `False` indicates the input times are unwrapped and will extend beyond 604800 for a new week. If the `wrapped` option is not specified, it defaults to `False`. Here's a sample pipeline:

{% highlight json %}
[
    "input.las",
    {
        "type": "filters.gpstimeconvert",
        "conversion": "ws2gst",
        "start_date": "2020-12-12",
        "wrapped": "True"
    },
    "output.las"
]
{% endhighlight %}

## Last Thoughts
Based on the amount of verbiage above, correctly converting between GPS time standards requires getting into the weeds and exercising some care. Hopefully, PDAL's `filters.gpstimeconvert` makes it a bit easier.

