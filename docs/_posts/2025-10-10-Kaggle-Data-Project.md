---
date: 2025-10-10 19:56:00
layout: post
title: "Kaggle E-Commerce Data Report"
author: giovannicalixte
subtitle: My initial findings from the E-commerce dataset...
description:
image: https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/E-commerce%20KPI%20Summary%20Dashboard-%20Top%20Locations.png?updatedAt=1761871722075
category: Project
tags:
- Data Analytics
- Marketing
- Tableau
- PowerBI
- Forecasting 
paginate: true
---

# Project Overview

The goal of this project was hone my data skills through practical application as opposed to relying on online tutorials. It also introduces me to concepts and knowledge I would need for data analytics jobs that required knowledge in marketing and forecasting. I think this is a post that I can revisit in the future as their are several parts I would like to refine. But as it is now I think it is worth sharing. The project is broken down as follows:

* Use PostgreSQL tools to normalize and model data as well as calculate key performance indicators (KPI's) relevant to retailers to aid Marketing.
* Use normalized data tables to build and publish dashboards in Tableau and compare it to PowerBI to get accustomed to the quirks of each software.
* Validate scalability of the dashboard by passing a 10-20 million row dataset into the dashboard.
* Use Python analytic tools to forecast total profit over time, learning best practices for applying predictive models to time-series data.

The dataset that was used for the analysis was the E-Commerce Data obtained from Kaggle. It contains data from a UK-based online retailer that does business in multiple countries, with the data on transactions occurring between 01/12/2010 and 09/12/2011, and the customers primarily being wholesalers. For this project, I will reveal the top and bottom spender by customer, country, and per capita. Customer purchasing frequency, as well as the following KPIs: churn rate, average order value, average purchase frequency, average customer lifetime, and finally customer lifetime value. I will also attempt to forecast the total profit using both linear and non-linear models.

[UPDATE](#update---10302025): This post was update with new guidance that expedites some steps.

## Normalizing and Modeling Data in PostgreSQL

Goal: Break original dataset into several data tables to normalize and practice data integrity. The following set of code creates new tables in PostgreSQL, setting the Primary Keys and Foreign Keys, and inserts the data from a staging table that holds all of the data:

```sql
-- Creating the table to hold all the data. Data is inserted into the table using the \\copy function in the PSQL client facility:
CREATE TABLE staging_orders (
    invoice_no VARCHAR,
    stock_code VARCHAR,
    description TEXT,
    quantity INT,
    invoice_date TIMESTAMP,
    unit_price NUMERIC,
    customer_id VARCHAR,
    country VARCHAR
);
```

```sql
-- Creating normalized data tables 
CREATE TABLE customers (
	customer_id VARCHAR PRIMARY KEY,
	country VARCHAR
);

CREATE TABLE products (
	stock_code VARCHAR PRIMARY KEY,
	description VARCHAR
);

CREATE TABLE orders (
	invoice_no VARCHAR,
	stock_code VARCHAR REFERENCES products(stock_code),
	quantity INT,
	invoice_date TIMESTAMP,
	unit_price NUMERIC,
	customer_id VARCHAR REFERENCES customers(customer_id)
);
```

```sql
-- This inserts data from the original source and cleans it of duplicates simultaneously
INSERT INTO customers (customer_id, country)
SELECT customer_id, MIN(country) AS country
FROM staging_orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
ON CONFLICT (customer_id) DO NOTHING;

INSERT INTO products (stock_code, description)
SELECT DISTINCT UPPER(stock_code), MIN(description) AS description
FROM staging_orders
WHERE stock_code IS NOT NULL
GROUP BY stock_code
ON CONFLICT (stock_code) DO NOTHING;

INSERT INTO orders (invoice_no, stock_code, quantity, invoice_date, unit_price, customer_id)
SELECT invoice_no, stock_code, quantity, invoice_date, unit_price, customer_id
FROM staging_orders
WHERE customer_id IS NOT NULL
  AND stock_code IS NOT NULL;
```

Once these tables were set up, I saved them using the PSQL client facility in to "orders.csv", "products.csv", and "orders.csv" respectively. I went on to calculate the top spenders in SQL as well as spending frequency per customer. Here is the code:

```sql
-- Top Customer By Spend
SELECT
	customers.customer_id,
	SUM(quantity * unit_price) AS total_spent,
	RANK() OVER(ORDER BY SUM(quantity * unit_price) DESC) AS spending_rank
FROM orders
INNER JOIN customers on orders.customer_id = customers.customer_id 
GROUP BY customers.customer_id
LIMIT 10;
```

```sql
-- Code for creating SpendFrequency.csv
-- Create CTE containing a query that returns all customer orders and the previous orders
WITH customer_orders AS (
	SELECT
		customer_id,
		invoice_date,
		LAG(invoice_date) OVER(PARTITION BY customer_id ORDER BY invoice_date) AS prev_order_date
	FROM orders
	)		
-- Query for returning the average interval between orders, but more importantly that interval in seconds as a numeric 
SELECT 
	customer_id,
	AVG(invoice_date - prev_order_date) AS avg_days_between_orders,
	EXTRACT(EPOCH FROM AVG(invoice_date - prev_order_date)) AS num_seconds
FROM customer_orders
WHERE prev_order_date IS NOT NULL
GROUP BY customer_id
ORDER BY num_seconds ASC
``` 
The spending frequency table returned by the last query above is ultimately saved into a "SpendFrequency.csv" file using the PSQL client facility and reloaded into a "spend_frequency" table that was created with this code:

```sql
CREATE TABLE spend_frequency (
	customer_id VARCHAR REFERENCES customers(customer_id),
	avg_days_between_orders INTERVAL,
	num_seconds NUMERIC
);
```
This sets up the environment for calculating all of the KPIs of interest for this project.

### Calculating KPIs in PostgreSQL

To restate the KPIs of interest, we are looking for: churn rate, average customer lifetime (ACL), average order value (AOV), average purchase frequency (APF), and finally customer lifetime value (CLV). The last three are easily calculated with the formulas readily available upon doing an online search. It becomes a bit more tricky when it comes to churn rate and ACL due to how granular you can get in calculating these values. It's good to be aware of this, but for the sake of expediency, in this project I calculate churn rate by dividing the number of customers that no longer shop at the retailer after the first 6 months by the total number of customers in that first 6 months, multiplied by 100 to convert to percentage and divided by 6 (for the 6 month period) to reveal the monthly churn rate. We use a rule of thumb to calculate ACL where it is just the reciprocal of the churn rate. The other KPIs - the main one being CLV - follows from there:

```sql
-- Once all tables are created, use this script to calc churn rate, AOV, APF, ACL, and CLV.
WITH cutoff_date AS ( -- period over which churn is determined in months and the cut-off date
	SELECT
		FLOOR(EXTRACT(DAY FROM (MAX(invoice_date)-MIN(invoice_date)))/2/30.5) AS churn_interval,
		MIN(invoice_date)+FLOOR(EXTRACT(DAY FROM (MAX(invoice_date)-MIN(invoice_date)))/2)*INTERVAL '1 day' AS middle_date
	FROM orders
	),
starting_customers AS ( -- having determined an interval to use as a cutoff find starting customers
	SELECT customer_id
	FROM orders
	GROUP BY customer_id
	HAVING MIN(invoice_date) < (SELECT middle_date FROM cutoff_date)
	),
churned_customers AS ( -- find customers that no longer have purchases after the cutoff
	SELECT customer_id
	FROM orders
	GROUP BY customer_id
	HAVING MAX(invoice_date) < (SELECT middle_date FROM cutoff_date)
	),
calc_churn_rate AS ( -- calculate churn and account for several months long period
	SELECT 
		(COUNT(churned_customers.customer_id)::FLOAT)/(COUNT(starting_customers.customer_id)::FLOAT)/
		(SELECT churn_interval FROM cutoff_date)*100 AS churn_rate
	FROM churned_customers
	FULL OUTER JOIN starting_customers ON starting_customers.customer_id = churned_customers.customer_id
	),
company_AOV AS ( -- Average Order Value
	SELECT ROUND(SUM(quantity*unit_price)/COUNT(invoice_date),2) AS AOV
	FROM orders
	WHERE quantity > 0 
	),
company_APF AS ( -- Average Purchase Frequency
	SELECT COUNT(invoice_no)/COUNT(DISTINCT customer_id)::numeric AS APF
	FROM orders
	WHERE quantity > 0
	),
company_ACL AS ( -- Average Customer Lifetime
	SELECT ROUND((1/(SELECT churn_rate/100 FROM calc_churn_rate))::numeric, 2) AS ACL
	FROM calc_churn_rate
	)
-- KPIs. 
SELECT
	(SELECT AOV FROM company_AOV) AS AOV,
	(SELECT APF FROM company_APF) AS APF,
	(SELECT ACL FROM company_ACL) AS ACL,
	ROUND((SELECT AOV FROM company_AOV)*
		(SELECT APF FROM company_APF)*
		(SELECT ACL FROM company_ACL),2) AS CLV
FROM customers
GROUP BY 1;
```

That is the end of the SQL portion of the project until a bit later where we use a script to simulate a 10M row dataset to validate scalability of the upcoming dashboard section.

## Visualizing E-commerce Data in Tableau and PowerBI

Goal: Refine visualization skills by making the same dashboards in Tableau and PowerBI, comparing the pro's and con's of both in doing so.

### Tableau dashboard (Most profitable locations, Customer behavior, KPIs - Gains, Losses, and Lifetime Value)

Instead of yapping about the creation process, I will share the Tableau dashboard right away (but will highlight key points afterwards):

<div class="dashboard-container" markdown="1">
  <!-- The dashboard HTML goes here -->
<tableau-viz id="tableauViz" src="https://public.tableau.com/views/E-CommerceDashboard_17593031296730/E-commerceKPISummaryDashboard?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link" hide-tabs toolbar="bottom"></tableau-viz>
</div>

Not the most prettiest of dashboards compared to what I've seen in the Tableau community, but it will get there. Some points to mention:
* The Tableau UI is much more appealing to look at in my opinion. Though it isn't always easy to figure out where things are located, it feels nice to navigate and create visualizations. It is kind of like Tableau is I-Phone and PowerBI is Android. 
* Calculated fields and set actions are crucial for implementing drill-downs in Tableau. The former is akin to DAX in Power BI. 
* To calculate KPIs, knowledge of Tableau's order of operations and Level of Detail expressions (LODs) is needed to prevent the effects of dimension filters.
* You can't complete calculations when some terms are aggregated and others are non-aggregated. This is where LODs and functions like ATTR() become key. 

### KPI dashboard in PowerBI

PowerBI is another popular visualization tool that is used in data analytics, and though I prefer Tableau right now, I'm seeing more and more the positives of PowerBI. Here is the KPI dashboard I made in Tableau replicated in PowerBI:


<img src="https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/PowerBIDash.png?updatedAt=1761867572939">

Unfortunately, I don't have the license that would allow for me to embed an interactive version of the PowerBI dashboard, so we will have to settle for the image here. Maybe I can get away with sharing the pbix file on the repo for this project? I also couldn't find a good way of re-creating the customer order frequency raster. I will provide an update once I do. Anyways, takeaways:

* As mentioned above, PowerBI's DAX is similar to Tableau's calculated fields in that it lets you compute constants and columns that can filter with dashboard interaction.
* Having used DAX a bunch now, DAX feels more intuitive to use when it comes to accomplishing control of level of detail than the syntax in Tableau. Level of detail can be controlled through the use of the CALCULATE() function in tandem with the ALL() family of functions. ALL() directly prevents filters from effecting values in calculations.
* Drill downs are much easier to set-up in PowerBI. They are more straight forward at least. 
* Not only does DAX feel like a more powerful tool that allows for much more complex calculations, PowerBI also has a Python scripter built in giving you full access to transform your data with all the packages you need, allowing more possibilities.

## Validating Scalability

Goal: Show that the Dashboard still performs well under the load of a much larger dataset.

### Simulating a 10M+ row dataset with E-Commerce data

Scalability seems to be a big buzzword in the tech industry for a while now. Though this isn't a first experience in dealing with issues of scale, it is a good and quick demonstration of it. In order to simulate a dashboard under the load of "big data", the "orders" table in the dataset was replicated into a newly created data table, "orders10M", several times in SQL until there were over 10 million rows in that table. Here is the PostgreSQL code for that:

```sql
-- This code replicates the data an arbitrary number of times.
DO $$
BEGIN
   FOR i IN 1..5 LOOP  -- repeat 5 times
      INSERT INTO orders10M (invoice_no, customer_id, stock_code, invoice_date, quantity, unit_price)
      SELECT 
          (invoice_no::INT + (SELECT MAX(invoice_no::INT) FROM orders10M WHERE invoice_no ~ '^[0-9]+$'))::VARCHAR,
          customer_id,
          stock_code,
          invoice_date + (INTERVAL '1 day' * (random() * 365)::int),
	  quantity + (random() * 3)::int,        -- vary quantity by up to +3
	  unit_price
      FROM orders10M
	  WHERE invoice_no ~ '^[0-9]+$' AND customer_id NOT IN (
	  	SELECT customer_id FROM orders GROUP BY customer_id
		HAVING MAX(invoice_date) < (
			SELECT (MIN(invoice_date) + EXTRACT(DAY FROM (MAX(invoice_date)-MIN(invoice_date))/2) * INTERVAL '1 day')
			FROM orders));
   END LOOP;
END $$;

SELECT COUNT(*) FROM orders10M;
```
Some key points to note with this code:
* New orders are randomly chosen from the orders10M table, with the order randomized in quantity and in date up to a year past that order date. Since the orders10M data is growing with each iteration, this means that the order dates grow forward in time - well pass the original dataset's last order.
* The orders that are randomly selected are required to have invoice numbers that can completely be converted to an **INT** type, as they are incremented with each iteration. 
* New orders were only added to customers who did not churn in the 6 month period that we originally chose in the first dataset. In other words, new orders were added to customers who remained loyal past the first 6 months in the dataset. This maintains the original data's churn rate and average customer lifetime, serving as a check.

Once the orders10M table was completed, it was used to re-calculate the spend_frequency table for the 10M+ orders. The new tables were exported as _orders10M.csv_ and _SpendFrequency10M.csv_ using the PSQL client facility.

### Data model and PowerBI KPI Dashboard for the 10M row dataset

Here is the overall data model in PowerBI, including each iteration of the data tables:

<img src="https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/EcommerceDataModel.png?updatedAt=1761184226589"> 

As seen in the image, the 10M row data was able to be integrated into the PowerBI data model alongside the original data without issue. Admittedly, the data model in the image also shows columns and measures calculated in DAX beforehand. These won't be present in a blank report and will need to be re-derived. 

Once the necessary columns and measures are derived for the 10M row datasets, the KPI dashboard seen previously can be re-created:

<img src="https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/PowerBIDash10M.png?updatedAt=1761867445715">_NOTE: Churn and ACL remain the same due to their relationship and how the data was scaled. Only AOV, APF, and CLV changed._

Churn rate and average customer lifetime is preserved.

## Forecasting Total Profit

Goal: Forecast total profit of the E-commerce data using time-series forecasting models in Python to gain familiarity and learn best practices.

### Forecasting with the ARIMA model

ARIMA stands for Auto-Regressive Integrated Moving Average and it is a gold standard tool for forecasting linear models. The auto-regressive (AR) component indicates that part of the model uses a weighted sum of previous values of the output up to a certain number of "lags" with error to predict new values. Similarly, the moving average (MA) component indicates that the mean of the signal and the weighted sum of previous residual errors from the mean up to a certain number of lags are used to predict new values of the data. Finally the integrated (I) component indicates that the time series undergoes a simple difference in order to ensure stationarity. 

This model can be used in Python through the statsmodels package, and so using the Pandas library, the E-commerce data can be loaded and aggregated in python to return the sum of the order totals on each day that exists in the dataset, creating the total profit time-series:

<img src="https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/TotalProfitSeries.png?updatedAt=1761184226675"> 

 A requirement in using ARIMA for forecasting is that the data that you are forecasting needs to exhibit stationarity in the mean of the signal as well as the variance. This means that:
* The mean doesn't change with time i.e positive or negative trends are removed.
* The variance or "seasonality" does not change with time.

The model uses techniques like maximum likelihood estimation to fit a probability distribution to the data that determine the "weights" on each lag in the model, so having a constant mean and variance is necessary to achieve accurate results. If the data does not exhibit these properties, differencing can be used to ensure stationarity of the mean of the time-series and transformations like a logarithmic or boxcox transformation can provide stationarity in the variance. These transformations exist within the scipy python library. For the E-commerce data we use a boxcox transformation:

<img src="https://ik.imagekit.io/ol32yu856/25_10_10_Kaggle_Ecommercie_Data_Post_images/BoxcoxSeries.png?updatedAt=1761184226595">_Boxcox transformed data. Can't really see the effect but it is there._ 


The last thing needed by the ARIMA model are the "orders" of each component of the model. The quantity provided for the orders of the AR and MA component determines the number of previous lags included in the sums for both components, while the quantity for the order of the I component determines how many times the time-series is differenced. For linear signals, this order is usually set to 1. But to determine the AR and MA orders, a partial auto-correlation function (PACF) and auto-correlation function (ACF) needs to be solve, respectively. For the E-Commerce data this looks like:

<img src="https://ik.imagekit.io/ol32yu856/Post_images/ACF_PACF.png?updatedAt=1760124916733">_ACF settles after 25 lags, PACF settles after 5 lags_

Orders are typically selected where each function no longer shows significance past the zero crossing or in this case the confidence band in blue. A more objective estimation of the orders can be accomplished by iterating through different selections of parameters and comparing the model score against Akaike's Information Criterion or Bayesian Information Criterion. The auto.arima function in the pmdarima package is able to do this for you. 

Once the model parameters are determine, they can be passed into the ARIMA function alongside a training portion (first 80%) of the time-series data to fit and create the model. The model is then used to forecast the portion that was excluded and finally an inverse boxcox transformation is applied to reverse the transformation.

### Forecasting with Meta's Prophet model

Data Scientists from Meta have developed a forecasting model that is considered a non-linear regression and is ready to use upon import. It just requires the data to be a dataframe containing **ds** as the name of the date column, and **y** to be the name of the time-series. No specialized knowledge is needed as the model is robust to missing data, outliers, trends, and seasonality. So to compare models, we quickly include Prophet to see how it performs with the data.

### Results  

Here is a plot of the forecast provided by each model:

<div style="max-width: 800px; margin: 0 auto;">
{% include 2025-10-10-Kaggle-Data-Project/Forecasts_ARIMA_Prophet.html %}
</div>

They both don't do too well, but Prophet without any extra tuning seems to do relatively better at the prediction than ARIMA. But in this case that make senses:
* The data seems non-linear to begin with, with a seasonality that isn't obvious at the day level of granularity. Having more data across a longer period of time and aggregated as such would probably help the ARIMA model.
* The data is also not continuous in time; There are discontinuities due to days where no orders were made. ARIMA seems to require data without gaps in time or is uniformly sampled, in other words. Prophet was built to work with daily data and should be robust to gaps in time, but more research is needed to see if any tuning can be applied there.

Using data at the day level of detail was chosen because it provided more samples in time. Aggregating to the month level created a time series that was very non-linear and consisted of less than 15 data points. As a check of the ARIMA code, we used the same code to forecast another time-series that is well known:

<div style="max-width: 800px; margin: 0 auto;">
{% include 2025-10-10-Kaggle-Data-Project/AirPassengers_Forecasts.html %}
</div>


### Conclusion/Afterword

**Initial insights:**
+ Though local customers make up a majority of the retailer's business, they are not the highest spenders on a per customer basis. 
+ Locals may benefit from local discounts; Internationals may pay additional fees for products, or behave in ways that increase amount spent. 
+ Further analysis can look into regional preference of products, their performance over time, and predictions of future trends in purchasing.
+ A month-to-month churn rate should be calculated as well. Also, a cohort analysis can be used to determine locations/areas of growth. 

A take away from this project are the realizations of the differences between Tableau and PowerBI as well as what good dashboard design looks like. Tableau has a much more appealing interface in my opinion and being able to publish the dashboard online and embed it anywhere should be a norm in the realm of visualization tools. However, I do like how features are more intuitive to implement in PowerBI and that it provides powerful tools for transforming data. Both do have their tiny quirks that can be frustrating and so neither tool is the perfect blend of what a "viz-ard" would want. But at the end of the day, I appreciate Tableau for letting us publish and share what we've built, license-free. 

The way I built the dashboard for this project harkens back to how I designed slide presentations while I was in graduate school. But looking at what the Tableau community produces, I realized that that isn't necessarily what the industry wants out of a true dashboard. This is something I knew subconsciously, as I've worked with dashboards without realizing it all my life owning a computer. Typically, true dashboards use a lot more space and make use of "furniture" that organizes the information into buckets depending on how they relate. They aren't necessarily a slide show but they do allow for navigation to other similarly built interfaces. My dashboard is pretty cluttered for the space and doesn't use furniture to separate information. I'll give myself some grace as the design mimics what I've built for classroom exercises that did make me aware of dashboard furniture and other design elements. Future dashboards I create will keep these design concepts in mind so that I can truly create something that is considered a dashboard.

Another big takeaway was the need to understand your data and the assumptions behind a predictive model when in the process of selecting a model to be trained by your data. If the data doesn't meet the requirements for the model, transformations need to be applied to the data to get it to obey those requirements. Otherwise the results of the predictions won't match expectations. The same process is applied when implementing machine learning algorithms, making the entire concept lose some of it's luster. I envision that one day it won't be just a matter of preparing the data and choosing a model. I want to be on the side of the field that creates or at least augments existing models, but that takes time and understanding more suitable for research roles and wouldn't fly in most high speed industrial settings. 

This project became really rewarding early on in the process. I started towards the end of September and I want to say I've been working on it daily. It is a project that I think I will revisit in the future to refine some parts like the dashboards, do some benchmarking on performance, and add extra analysis like churn rate over time. But I think where it is right now is a good stopping point. I will definitely update this post in the future.

I already have ideas for the next project. It will be smaller in scope, but quicker to put out. I soon want to dig my teeth into more traditional machine learning projects and with other kinds of datasets. But as with most things, I need to take it a step at a time.

Thanks for viewing this post!

## UPDATE - 10/30/2025 

After looking at other analysis of this data set, I realized some of the quantities needed to be recalculated, code required some updating, and my dashboards needed a redesign. Here are the changes:
+ Previous AOV, APF, and consequently CLV were calculated on a per customer basis. They all should be an aggregate value taking all customers into account. This creates a single metric that you can score a company's performance against. The SQL code and formulas in the Dashboards have been updated to reflect this.
+ Because of the above, it turns out the SpendFrequency*.csv files are no longer needed for any calculation and so they can be removed from the data model and the SQL code can be ignored.
+ The main visual of this post is no longer relevant, and so I took the time to redesign the entire dashboard to fill the gap. I learned new techniques in doing so and utilized the concept of furniture in it's design. It looks better than what I had before and reflects what I've seen in the Tableau community. I think I can be more minimal though and would like to add more variety to the visuals. 

Giovanni

_All of the files generated except the dashboards and the datasets related to the 10 million row datasets can be found in my GitHub repository located [here](https://github.com/GcalGit93/ECommerceDataAnalytics)._

