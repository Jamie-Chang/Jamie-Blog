Title: My Chinese birthdays, time keeping is hard!
Date: 2025-06-25
Category: Blog

I grew up in both the UK and China. So I'd like to think I have a little understanding of both cultures. China uses both the western gregorian calendar and a version of lunar calendar called 农历. This means I have a Chinese lunar birthday as well as a western one. 

I did find a small library called [lunardate](https://github.com/lidaobing/python-lunardate):

<link rel="stylesheet" href="https://pyscript.net/releases/2025.5.1/core.css">
<!-- This script tag bootstraps PyScript -->
<script type="module" src="https://pyscript.net/releases/2025.5.1/core.js"></script>

<script type="py-editor" config='{"packages":["lunardate"]}'>
    from lunardate import LunarDate
    print("I was born on:", LunarDate.fromSolarDate(1995, 6, 28))
    print("My birthday this year is on:", LunarDate.fromSolarDate(2025, 6, 25))
</script>
> Don't go all out on presents though, there's another one in exactly a month. 

If you run `LunarDate.fromSolarDate(2025, 7, 25)` you get `LunarDate(2025, 6, 1, 1)` which is the same date as `LunarDate.fromSolarDate(2025, 6, 25)` barring the last parameter `1` which indicates a leap month. Unlike leap days a leap month is a 'repeated' month. Happy birthday to me again! 

> A leap month is caused by the difference between a solar year (365 ish days) and a lunar year 354 ish days. 3 months are added every 19 years. The maths checks out but the exact mechanism depends on some astrological observations.

Fun fact, I wondered how the library knows when to add a leap month and the answer is quite simple! It has simply stored [all leap months](https://github.com/lidaobing/python-lunardate/blob/master/lunardate.py#L351-L405) between 1900 and 2099.


## Back to the point
> Apologies for the long winded story about my extra birthdays, now back to the regularly scheduled Python content

I started thinking about how the chinese calendar is complicated syncing moon cycles to solar years. Trying to sync two cycles that don't directly relate to each other has necessitated the concept of leap months. 

But then it occurred to me that's exactly what almost all calendars are doing! Leap days itself are necessitated by the ~8hr error trying to synchronise days with years. And gregorian months are really all over the place. Timekeeping in Python and other languages also has its quirks.

### Leap Seconds
Leap months are actually a very similar concept to [leap seconds](https://en.wikipedia.org/wiki/Leap_second). Where occasionally due to various drifts in our time keeping we add an extra second. Like leap months, the date is determined by measurements, which is to say it's not deterministic when the leap second will be added. Though there is usually half a year of warning before the introduction. 

In Python as in many programming languages and platforms, leap seconds are not explicitly tracked or represented due to its non-deterministic nature. Operating systems will either repeat the second deliberately slowdown the clock to accommodate for the extra second.

Though it's just a single second, this does have the ability cause [issues](https://www.wired.com/2012/07/leap-second-glitch-explained/). In python to track a leap second you can use [astropy](https://docs.astropy.org/en/stable/time/index.html) which accounts for leap seconds

<script type="py-editor" config='{"packages":["astropy"]}'>
from astropy.time import Time, TimeDelta
import astropy.units as u

# Define two dates that span a leap second
# A leap second was added on 2016-12-31 at 23:59:60 UTC
t1 = Time("2016-12-31T23:59:59", scale='utc')
t2 = Time("2017-01-01T00:00:01", scale='utc')

# Calculate the duration
print(f"{(t2 - t1).sec} seconds")
</script>

Whilst it's unlikely that you'll need to regularly reach for this, it is good to know what to do when the issue of leap seconds becomes relevant.

### Datetime and timezones
> NOTE: the pyodide editors may not load timezones properly, apologies in advance

I get reminded (almost) every year on new quirks about it. For the most part the only reasonable thing to do is to use UTC whenever possible:


```py
datetime.now(UTC)
```

if you use `datetime.now()` without timezones please don't, it's bad, you'll regret it sooner or later. `datetime.utcnow()` is really not much better and has in fact been deprecated.

Now just because you use timezone aware datetime doesn't mean you're safe. If you use a timezone with daylight saving time, when the times go back 1hr we get an overlapping interval (a folded datetime):

<script type="py-editor" env="datetime" config='{"packages":["tzdata"]}' setup>
from datetime import UTC, datetime, timedelta
from zoneinfo import ZoneInfo
LONDON = ZoneInfo("Europe/London")
</script>


<script type="py-editor" env="datetime">
from datetime import UTC, datetime, timedelta
from zoneinfo import ZoneInfo

LONDON = ZoneInfo("Europe/London")
dt = datetime(2025, 10, 26, 1, tzinfo=UTC)
dt.astimezone(LONDON)
</script>
This returns `datetime.datetime(2025, 10, 26, 1, 0, fold=1, tzinfo=zoneinfo.ZoneInfo(key='Europe/London'))`

<script type="py-editor" env="datetime">
datetime(2025, 10, 26, 0, tzinfo=UTC).astimezone(LONDON)
</script>
Returns the non-folded `datetime.datetime(2025, 10, 26, 1, 0, tzinfo=zoneinfo.ZoneInfo(key='Europe/London'))`

So far so good, but what if we compare them like this:

<script type="py-editor" env="datetime">
(
    datetime(2025, 10, 26, 0, tzinfo=UTC).astimezone(LONDON) 
    == datetime(2025, 10, 26, 1, tzinfo=UTC).astimezone(LONDON)
)
</script>
We get `True` which is not at all what I expected. This is because for same zone comparison only the wall clock time is used to preserve backwards compatibility, you can read more about it in [PEP-495](https://peps.python.org/pep-0495/#backward-compatibility)

But the weirdness doesn't end here: 
<script type="py-editor" env="datetime">
(dt := datetime(2025, 10, 26, 1, tzinfo=UTC)).astimezone(LONDON) == dt
</script>

Normally the above script will return `True` for any datetime, DST or otherwise. But during folded time (whichever side of the fold you're on), this will [always return False](https://peps.python.org/pep-0495/#aware-datetime-equality-comparison). 

The reason is is explained in the footnote:

>This exception is designed to preserve the hash and equivalence invariants in the face of paradoxes of inter-zone arithmetic

If `datetime(2025, 10, 26, 1, tzinfo=UTC)).astimezone(LONDON)` were to equal `datetime(2025, 10, 26, 1, tzinfo=UTC)` then the hash must be equal. But due to backwards compatibility the hash is not equal there. So we don't allow the value to be equal either. 

This is so baffling, it took me several tries to understand it.


## Finally
Honestly I learned a lot more about time keeping than I expected when researching this topic. I believe no matter the language or tools, it will always be a difficult problem. And as such, we must treat it with the care it deserves when we're building a system that is sensitive to these timestamps.