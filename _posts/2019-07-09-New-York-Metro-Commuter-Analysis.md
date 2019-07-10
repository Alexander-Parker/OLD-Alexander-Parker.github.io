---
layout: post
title:  "New York Metro Commuter Analysis"
date:   2019-07-09 12:00:00
categories: data visualization
---


I recently started at Metis, an intensive 12-week data science-focused bootcamp, at their Chicago campus. Our first project was to make sense of some MTA (New York City’s subway authority) commuter data on behalf of our “client” who was hoping to promote an upcoming women-in-tech focused gala at a select number of subway stations.

The questions we wanted to answer for our client were not just which stations would have the most commuters going through them, but which stations would be best to target based on activity weighed against the demographic profiles of the stations’ commuters. It would also be important to provide our client with information about when to target certain stations, such as what time of day or what day of week. 

## Gathering the Data
Answering which stations were busiest (which had the highest average commuters) was the most straightforward component of our question and the focus of our initial data collection efforts. We relied primarily on [MTA turnstile data](http://web.mta.info/developers/turnstile.html) for this. This dataset shows the cumulative entries and exits at each turnstile (designated as a unique control area plus specific turnstile ID) over what were generally four-hour intervals. We performed a few cleaning operations on the dataset:
*	Bucketed observations into six standardized four-hour intervals over the day
*	Converted entries/exits to reflect entries/exits for the period rather than cumulative totals
*	Eliminated observations with aberrant entries/exits (no entries/exits or an improbably large number of entries/exits)
*	Outputted an intraday (four-hour interval) and combined daily datasets of entries/exits for each control area/turnstile combo

We also wanted to attach some coordinate data to our entries in order to plot the stations on a map. We used the [MTA’s station locations csv](http://web.mta.info/developers/data/nyct/subway/Stations.csv) to grab the latitudes and longitudes. Unfortunately, station names do not match between the turnstile and station locations data. We utilized a hand-matching done [here](https://groups.google.com/forum/#!searchin/mtadeveloperresources/control%7Csort:date/mtadeveloperresources/UjbdfrW0nWM/Ky9k7OkUCAAJ) of control areas to station IDs along with fuzzy string matching to attach latitudes and longitudes to control areas with no hand match. 

We gathered additional data to reflect potential demographic traits of our commuters. We were primarily concerned with giving weight to the percentage of women utilizing a station. We also wanted to consider students/professors who may have direct interest or connections to technology fields. We incorporated gender by gathering the percentage of women in the population by zip code from the 2010 census. This was linked to specific stations by reverse geocoding stations’ coordinates via the google geocoding API and facilitated by the geocoder python package. 

## What Stations to Target? A Simple Ranking of Stations
Now that our data was gathered and ostensibly cleaned, we came up with some simple metrics to compare stations. We decided on three metrics to look at initially:
*	Daily average commuters (including entries and exits)
*	Daily average commuters multiplied by the percentage female population of a station’s corresponding zip code
*	A count of universities in close proximity to a station (defined as within 3/4s of a mile), bucketed into four categories: High, Medium, Low and None. 

The table below shows the top ten stations by daily average commuters. Along with their respective demographic-focused metrics. We found that there was little variation in these demographic metrics and believed it was important to focus on the largest of stations as they were significantly larger than their peers (see below for a swarm plot of station observations – the distribution of average daily ridership is very right skewed). 

<table>
<thead>
    <tr>
        <th style="text-align: right;">Rank</th>
        <th>Station</th>
        <th style="text-align: right;">Avg. Daily Commuters</th>
        <th style="text-align: right;">% Female Commuter Rank</th>
        <th>University Exposure</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td style="text-align: right;">1</td>
        <td>34 ST-PENN STA </td>
        <td style="text-align: right;">259,612</td>
        <td style="text-align: right;">5</td>
        <td>Medium</td>
    </tr>
    <tr>
        <td style="text-align: right;">2</td>
        <td>34 ST-HERALD SQ</td>
        <td style="text-align: right;">186,832</td>
        <td style="text-align: right;">367</td>
        <td>Medium</td>
    </tr>
    <tr>
        <td style="text-align: right;">3</td>
        <td>GRD CNTRL-42 ST</td>
        <td style="text-align: right;">180,043</td>
        <td style="text-align: right;">3</td>
        <td>Low</td>
    </tr>
    <tr>
        <td style="text-align: right;">4</td>
        <td>14 ST-UNION SQ </td>
        <td style="text-align: right;">171,547</td>
        <td style="text-align: right;"> 6</td>
        <td>High</td>
    </tr>
    <tr>
        <td style="text-align: right;">5</td>
        <td>TIMES SQ-42 ST</td>
        <td style="text-align: right;">164,358</td>
        <td style="text-align: right;">7</td>
        <td>Medium</td>
    </tr>
    <tr>
        <td style="text-align: right;">6</td>
        <td>23 ST</td>
        <td style="text-align: right;">147,124</td>
        <td style="text-align: right;">1</td>
        <td>High</td>
    </tr>
    <tr>
        <td style="text-align: right;">7</td>
        <td>42 ST-PORT AUTH</td>
        <td style="text-align: right;">146,540</td>
        <td style="text-align: right;">11</td>
        <td>Medium</td>
    </tr>
    <tr>
        <td style="text-align: right;">8</td>
        <td>86 ST</td>
        <td style="text-align: right;">131,358</td>
        <td style="text-align: right;">4</td>
        <td>None</td>
    </tr>
    <tr>
        <td style="text-align: right;">9</td>
        <td>FULTON ST</td>
        <td style="text-align: right;">126,048</td>
        <td style="text-align: right;">2</td>
        <td>Medium</td>
    </tr>
    <tr>
        <td style="text-align: right;">10</td>
        <td>125 ST</td>
        <td style="text-align: right;">119,683</td>
        <td style="text-align: right;">8</td>
        <td>Medium</td>
    </tr>
</tbody>
</table>

Our busiest stations were all located in Manhattan and include many recognizable names. We used the folium python package to map out the stations below:

<iframe src="public/html/static_map.html" height="500px" width="100%"></iframe>

## When to Target? A Bit More Detail on the Largest Stations
We wanted to be able to tell our client when to target certain stations along with which stations to target. We generated heatmaps to show ridership activity over several different timeseries in order to highlight time considerations. These heatmaps were generated in folium using the appropriately-named [HeatMapWithTime plugin](https://python-visualization.github.io/folium/plugins.html#folium.plugins.HeatMapWithTime). Note that these maps include all stations, not just our top ten. The map scale is specific to each time step of the map and should be used to interpret activity relative to other stations in the same time period rather than absolute activity across time periods

### Average Ridership by Day of Week
<iframe src="public/html/time_map_byday.html" height="500px" width="100%"></iframe>

### Average Ridership by Time of Day
<iframe src="public/html/time_map_intra.html" height="500px" width="100%"></iframe>

Manhattan stations showed different patterns of activity during the weekends versus work days. We also noticed that Manhattan stations were generally the busiest of all throughout the day by a large margin with the exception of early morning (4:00 – 8:00 AM), where Manhattan was relatively less busy than the rest of the subway network. 

We plotted the average activity of four-hour intervals for our top ten stations utilizing two cuts of our dataset. The first cut included only work days and the second included only Saturdays and Sundays. All stations followed similar patterns, with weekday activity concentrated on morning and evening rush hours and weekend activity generally picking up in the afternoon.

<img src="public/img/workday.svg" width="100%">
<img src="public/img/weekend.svg" width="100%">


## A Few Takeaways
The largest stations in Manhattan seemed to be the best targets of a promotional campaign given the disproportional number of commuters that go through these stations, even when weighing demographic concerns such as gender and education. These stations would see the largest volume of commuters during rush hour, but will still see smaller volumes on weekends when commuters are potentially amenable to discussing the event with a promoter. 

Our work on this project created a few additional questions that would be interesting to explore. We recommended stations that will likely handle a large number of tourists and other people in New York for a short period of time; not the best targets for a gala to be held in the city at a later date. The MTA also offers [fare data](http://web.mta.info/developers/fare.html), a weekly aggregation of fares grouped by method of payment, which could be used to better target specific groups of commuters. For example, tourists may be using short-term time passes versus residents who may have EZ pay set up or a metrocard. Student fare could also be considered as well. 