# MBTA Performance API

## Origins

Anyone who has ever commuted by subway (NY, DC, Boston, etc.) will have run into
mystery delays. Whether on the platform, or in a train, the delays are sudden,
and inevitably infuriating as the transit authority almost never gives riders
full information about the situation. The MBTA of Boston may indicate the cause
of the delay over their alert feed, and may describe the delays in vague terms
like "minor" or "moderate", but what do those terms actually mean? 

In a short two-week project to approach a data problem using good
object-oriented coding practices, I had the goal of quantifying the se
qualitative terms to their actual affects on riders' commutes. Or perhaps
modeling expected delays due to different weather conditions. After exploring
the data available from different transit authorities, I found that the MBTA of
Boston had a web API that yields historical train performance data. Nice, no
need to scrape real-time data for months on end! I thought it'd be 
straight-forward to jump right into the modeling after a little data
consolidation...

The data available turned out to be much more challenging to wrangle than
previously thought. I therefore shifted the goal of the project to be
simplifying data extraction from the MBTA API, and putting the data in a form
that would actually be useful during further analysis.

## Challenges

To understand the data challenges, let me first lay out the structure of the
data in the MBTA API. Fundamentally, as a train travels through the system, it
is in one of two states: stopped at a station, or travelling between stations.
By tracking all the stops and movements of the train through the system, you can
determine the time it takes to travel between any set of stops. The baseline
performance can be established from many trains at different times of the day,
and delays can be compared against the baseline.

In the MBTA API, the stop and travel infomation is not consolidated on a
per-train basis as would be desirable. It rather requires you to query the stops
at each station separately, using a station ID number defined in another API.
The travel times between stop pairs must also be queried individually.
Technically the API could be queried for the travel times between any stop pairs
along the line. This would yield an enormous amount of redundant data if you
already have data for the "nearest-neighbor" station pairs, and essentially
squares the amount of data that needs to be queried, stored, and analyzed. I
therefore restrict the travel data to the “nearest-neighbor” station pairs. In
the end, this yields a collection of train "events" in a number of separate files
that need to be combined together: for example on the Red line, a stop at Davis
is connected to a travel between Davis and Porter, connected to a stop at
Porter, connected to a travel between Porter and Harvard, etc. 

What makes this difficult is that only timing data is provided for the trains,
no train IDs. You therefore have to take the train that travels from Davis
to Porter, and match it to the same train that stops at Porter by matching
the time that the train arrived at Porter (provided in both the travel time
and stop time files). The primary challenge here is to efficiently do
this chaining if you have a dataset with thousands of possible matching
segments.

This procedure would be fairly trivial if the train data were fully populated.
Unfortunately, train data can be somewhat sparse between some station pairs for
some reason. This means that chaining will produce many train fragments that
ideally would be connected into single trains.

## Consolidation Strategy

To tackle the first challenge (chaining the right train segments efficiently and
correctly out of thousands of possible combinations), fortunately the MBTA API
yields time ordered train events. If you start with the earliest possible train
segment, you only have to look near the front of the list of train events in other
segments to do the matching. These matching segments can then be removed from
the list so you don't waste time later with these already-used segments. As this
is a largely pop-from-front process, a `deque` of train events in each track
segment and stop is used and makes the chaining process efficient. An idealized
schematic is shown below.

![image](figures/time data flow.svg)

Here, `t0_0` is the earliest available departure time, so that time will be
popped first. The following segments are iterated over, and matching times (in
red) are popped as well until a matching time is not found, or the end of
the line is reached. The next earliest train can then be constructed from the
remaining times.

Because of missing travel time information, only part of a train's total journey
can be consolidated through chaining, yielding a train fragment. Fortunately,
the same train typically "reappears" later in the line, and the two segments
just have to be connected again. A typical situation like this is shown below.

![image](figures/train combination.svg)

Several conditions are required to be met to connect the fragments:

1. The start of the second train must be later in the line than the end of the
first train.
2. The start and end must be less than four track segments from each other (it's
easy to connect data that's close temporally and spatially).
3. If the trains' start and end are one track segment from each other, the
travel time in that track must be in the range [0s, 180s]. The 0s lower bound is
allowed because sometimes the missing travel time is incorrectly added to the
stop time information from the previous stop.
4. If the trains' start and end are more than one track segment from each other,
the average travel time per track segment must be between [60s, 180s].

Because the previous trains have been produced in time-order, you simply have
to look at the back end of the list of trains that have been constructed for
another train matching these conditions. The search aborts if the trains
searched are too far back in time (1 hr between start and end time). 

## Project Structure

Using these strategies, I developed `mbta_performance` as a python-module that
allows an interested data scientist to easily pull data from the MBTA API, and
does the data consolidation to allow more straight-forward analyses to be
performed

The base class to allow an MBTA analysis is
`mbta_performance.train.TrainCollection`. This class will allow the user to
download subway performance data, compiled into individual trains. These trains
can then be used for further analyses (e.g. MBTA delay announcement
responsiveness, delay magnitude prediction based on weather, etc.).

To start an analysis, simply `import mbta_performance`, and load a line:
```python
tc = mbta_performance.train.TrainCollection ()
tc.load_base_train (mbta_performance.train.lines.<line>)
```
The train direction can also be set by the `direction_id` tag ("0" or "1").
Using `datetime` objects, train performance data can be downloaded from the MBTA
API by:
```python
tc.set_data_path (<directory to download data to>)
tc.get_times (start_datetime, end_datetime)
```

Once this is done, the obtained files can be loaded for analysis:
```python
tc.load_times ()
```

Based on these files, `Train` objects are created, representing the path of a
single MBTA train through the line:
```python
tc.load_trains (num_trains=<desired train collection size>)
```
These trains are available at `tc.trains`. Each `Train` (e.g. `t =
tc.trains[0]`) is a collection of `TrainStop` objects (`t.stops`) and
`TrainTrack` objects (`t.tracks`), which hold information on the time the train
encountered that segment of its journey. All trains in the collection can be
iterated through by either `for t in tc` or `for t in tc.trains`.
`TrainCollection` objects further support slicing, where `tc[:100]` would return a
`TrainCollection` with only the first 100 `Train` objects.

Please note that because of missing information, a single `Train` may not have
information across the entire line. The interval over which the Train has data
(not necessarily complete) can be assessed through `t.start` and `t.end`,
which point to the starting point and ending point of the train data. This
interval can easily be traversed by iterating over the train, for example:
```python
for p in t:
    ...
```
This will access all `TrainStop` objects and `TrainTrack` objects between the
start and end, in line-order (alternating stops and tracks).

A specific segment of a given train can be easily accessed from the usual
`python` slice syntax. For example, `t[3:8]` would yield a train defined between
the 3rd and 7th stop in the train's route.

Perhaps the single most important metric for a single train is its end-to-end
travel time. This can be obtained through `t.total_travel_time`. This will
return a `tuple` containing the total travel time in seconds at index `0`, the
train's starting stop at index `1`, and the ending stop index at `2`.

To visualize the performance of an ensemble of trains, please see
`TrainCollection.plot_trains`. This method results figures like:

![image](figures/Orange_travel_time.png)

Finally, and "average" train for the ensemble is accessible with
`tc.median_train`. This train contains the median travel times and dwell times
in each leg of the train's journey through the line.
`tc.median_train.total_travel_time` can then give a baseline estimate of the
total travel time through the system for further analysis.


