---
date: 2026-05-21 07:00:00
layout: post
title: "Airline Delay 15/25 Analysis"
author: giovannicalixte
subtitle: My insights from delay data from this past decade...
description:
image: https://ik.imagekit.io/ol32yu856/AirlineDelay2015_2025/Dashboard%203%20-%20Cropped%20.png?updatedAt=1778259946193
category: Project
tags:
- Data Modeling
- Data Analytics
- Python
- SQL
- Tableau 
paginate: true
---

_All of the files generated except the dashboards can be found in my GitHub repository located [here]()._


# Project Overview and Executive Insights (Summary)

<h2 id="AirlineDash"></h2>
<div class="dashboard-container" markdown="1">
  <!-- The dashboard HTML goes here -->
<tableau-viz id="tableauViz" src="https://public.tableau.com/views/2015_2025_AirlineDelay_Dashboard/Dashboard1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link" hide-tabs toolbar="bottom"></tableau-viz>
</div>

This project is meant to serve as another experience to refine my skills in completing an end-to-end analytics project using Python, SQL, and Tableau. The airline delay data was obtained from the U.S DOT's Bureau of Transportation Statistics website and spans the months starting in June 2015 through June 2025. The raw dataset contains information for flights on major carriers, their arrival airport, the total arrivals/diversions/cancellations of that carrier at that airport for that month, as well as their total delayed flights and minutes broken down by delay cause.

**Insights:**
+ _Delays throughout the decade were primarily due to carrier or late aircraft events with the latter being the primary cause._
+ _Overall, American Airlines and Southwest Airlines have the highest total delays by carrier, while Chicago O' Hare International and Dallas/Fort Worth International have the highest total delays by airport._
+ _Late aircraft and carrier delays saw the most impact during 2020 - most likely due to the impact of the global pandemic on customer behavior - reducing average flight delays by essentially an order of magnitude (~10%)._
+ _A reciprocal relationship between the two delays can be seen: carrier delay increased due possibly to less carrier availability during the pandemic but simultaneously this reduced late aircraft delays because there we less carrier's to deal with overall._
+ _The impact on delays in 2020 did not begin to return to what they previously were until about 3 years afterwards._
+ _Despite having larger average delays, Airports that serve a large amount of customers and flights still maintain around an 80% on-time rate, suggesting efficient workflows with smaller but many accumulated delays versus large, catastrophic delays._

## Python Exploratory Data Analysis and PostgreSQL Normalization - From Long to Wide, From Wide to Long

### Python

As with previous projects, a component of my analysis has been to used tools in python to get a sense of the data and to extract some insights with its specialized packages. This is in effort to learn and keep my skills sharp. In this project, I use python mainly to do exploratory data analysis, but the code is set up in a way where I can easily offload the curated data into a forecasting pipeline. I mainly wanted to become more familiar with the pandas toolkit - to be able to generate summary figures and metrics with it's native tools with very little assistance from other packages. 

The most useful tool I rediscovered from my classroom time working with pandas was the "unstack" function. It allows the user to pivot any column level created through "groupby" aggregations into a row, allowing the user to better see how an entry varies across columns and rows (one example would be columns representing airports, rows representing years, with the values where both meet being a delay time aggregated over that time span for that airport). It becomes more intuitive to access the data (in my opinion) as well as extract and/or plot all or slices of the data in a column vs row header format.

<img src="https://ik.imagekit.io/ol32yu856/AirlineDelay2015_2025/Unstack.png">_Pandas array goes from a "long" format with many rows with several sublevels, to a "wide" array by unstacking one of the levels._ 

The KPIs calculated in python were _on-time rate, cancel rate, average delay, and ranked delay causes._

### PostgreSQL

PostgreSQL was used for normalizing the raw dataset used in python into dimension and fact tables in a STAR schema that would be suitable for use in Tableau. With airline delay data, I initially normalized the data into carrier, airport, and flight date dimensions with a flight stats fact table. But since we also had a large amount of information on specific flight delays, I ultimately split the original fact table into two fact tables in what is called **fact table normalization** or **fact decomposition** - one fact table contains total arrivals, tardy, cancelled and diverted flights while another table contains specific flight delay minutes and causes attribution by count. This second fact table has it's own dimension table called the delay cause dimension which contains a serially generated ID number for each cause name inserted into the table.


Since there are 10 columns associated with the delay cause dimension, it was suggested that I further organize the flight delay fact table from a wide format to a long format with the help of **unpivoting**. The original unpivot I was looking at suggested using four UNION ALL commands across five SELECT statements - 1 for each delay type - to append the delay minutes and delay counts for each cause. I opted to use a CROSS JOIN LATERAL unpivot to practice a more elegant solution. Here is the code for that portion:

```sql
INSERT INTO fact_flight_delays (flight_id, delay_cause_id, delay_count, delay_minutes)
SELECT
	f.flight_id,
	dc.delay_cause_id,
	delays.delay_count,
	delays.delay_minutes
FROM stagingairlinedelay AS s
--- following joins serve to match up serially generated ids to original staging table based on carrier and airport codes as well as flight dates
JOIN carrier_dim AS c 
ON s.carrier = c.carrier_code

JOIN airport_dim AS a
ON s.airport = a.airport_code

JOIN fact_flights AS f
ON f.date_id = (s.delay_year*100+s.delay_month)
AND f.carrier_id = c.carrier_id
AND f.airport_id = a.airport_id
--- This will pivot the delay quantities so they become row entries for each delay going from wide to long 
CROSS JOIN LATERAL (
	VALUES
		('Carrier', s.carrier_ct, s.carrier_delay),
		('Weather', s.weather_ct, s.weather_delay),
		('NAS', s.nas_ct, s.nas_delay),
		('Security', s.security_ct, s.security_delay),
		('Late Aircraft', s.late_aircraft_ct, s.late_aircraft_delay)
) AS delays(cause_name, delay_count, delay_minutes)

JOIN delay_cause_dim AS dc
ON dc.cause_name = delays.cause_name

---WHERE delays.delay_minutes >= 0
---	OR delays.delay_count >= 0; --- this filter will actually filter for each delay cause in the value array. If the particular delay is greater than 0 for particular records, those records are the ones included in the pivot and joined
```  

The final data model is shown here:

<img src="https://ik.imagekit.io/ol32yu856/AirlineDelay2015_2025/AirlineDelays2015-2025Schema.png?updatedAt=1779056109172">_Airline Delay Star Schema_

For each time-grain, airport, and carrier, an average delay, an on-time rate, a cancellation rate, and a ranking of delay cause was computed within SQL. 

## Tableau - Key Design Notes

Here is a link back to this project's dashboard: <a href="#AirlineDash">Link to Tableau Dashboard</a>

Compared to [previous](https://gcalgit93.github.io/gcalgit93/project/2025/10/10/Kaggle-Data-Project.html#EcomDash) dashboards, a lot more involvement was put into stretching my design skills and leveraging the functions Tableau has to offer to make an appealing experience with this dashboard. Here are key techniques implemented for the major components:

* More aesthetic and dynamic **KPI Cards** - I utilized multiple marks with _placeholder axes_ (SUM(0)) and the _dual axis_ setting to create KPI cards with background designs representing an abstracted version of the KPI's time series. Exposure and the KPI are controlled by calculated fields and parameters. I apply the same techniques to the pie charts.
* **Simplified drill-downs** in the time-series chart using parameters and _IF-ELSE_ calculated fields. That dashboard is meant to be more for a technical team so I did not try to get too fancy in creating an appealing aesthetic. Many controls was the motto here.
* Conversely, I got fancy with the **YoY change** worksheet using only visuals to represent the magnitude of change. The _LOOKUP()_ function was helpful in creating the specific YoY table calculation I wanted here (comparing the calculations from values within the month 12 months prior).
* A **multi-functional map** - used the airport name field to generate coordinates for the map. Specified the circle then pie mark to get a scatter of pie charts showing delay break down for each airport with a tooltip insert of that airport's blown-up pie chart. Scatter points are scaled by different calculated fields reflecting a performance metric.
* A **"ghost" scatterplot** - uses the _dual placeholder axis_ technique to plot the same points on overlapping plots. One axis plots all points at a lower opacity and the other axis plots a filled in subset of points on top based on carrier ID. This achieves controlled highlighting of points.
* **Table calculation-based Top/Bottom-N Rank Chart** - you can't use the basic implementation of TopN filtering with fields that are table-calculations. My workaround was to create a calculated field to select entities above or below a certain rank based on the count of entities (Top and Bottom 10). The key with the bottom ten is that an LOD calculation that is then aggregated is needed to get the total count of entities to be comparable to the rank table-calculation. This method avoids hard-coding the cut-off for the bottom ranks.
* **Custom shape navigation buttons** made in Inkscape to add subtle aesthetics

## Conclusion/Afterword

It was important that I first explore the data via Excel and Python as I did not know anything about airline delay data going in even with clarification from the definitions spreadsheet. Once I got to plotting, a lot of the columns started to make sense. The columns that were the most unclear were the delay cause counts. A count should be an integer, but those columns had fractional digits. But all that is saying is that that fraction of the delay time is attributable to that cause. Totaling the delay counts should yield a whole number indicating the number of delay events for that flight in that month. Another reason why I started with Python here was to practice using Pandas more - to become comfortable with doing the majority of transformations within that framework as it seems that that is favorable for working with the machine learning tools in scikit-learn. I plan to make machine learning a central aspect of my next big project.

When I started creating the data model in SQL, I originally was going to make a traditional star schema with the flight information fact table and the airport, carrier, and flight date dimensions. That would have been fine, but coach ChatGPT suggested decomposing the fact table even further into general flight stats and flight delays and to unpivot the latter since it would be wide otherwise. Unpivoting certainly made the data model look cleaner, so that is a technique I will keep in my toolkit and apply where it fits. Besides the decomposition and unpivot, further data validation involved standardizing carrier names and removing missing values.

As for the dashboard, I really wanted to design something that looks nice and worthy of the Tableau community. This started with the KPI cards - the design tips being taken from the Golden Insights YouTube channel. It was also in the application of a consistent color scheme across the dashboards and the use of custom shapes for the navigation buttons. Some changes I'd make to improve parts of the dashboard would be to completely redesign the average delay time-series chart. It functions how I intended it, but the chart still looks clunky. I can envision a version that would still serve technical teams while still being visually appealing. I'll need to practice a bit more before I can implement that design.

In my last post, I mentioned that my next project wouldn't take so long to come out. It is several month later admittedly, but the project I originally intended to work on actually has a copyright on the data source and so I can't reproduce it here. I do have two other reports with repos that were private at the time of publishing this project. Those will become public very soon as they relate to my Ph.D. dissertation. I also - at the time of writing this - started working for Grady Memorial Hospital as a Junior Business Intelligence Developer. Because of that, my next big end-to-end analytics project will be a while away, but I should be much more skilled when I come around to it. I am aiming for that project to be more predictive model heavy and within healthcare. It will be larger in scope, for sure, but I will have several smaller supporting posts in the lead up to that project.




