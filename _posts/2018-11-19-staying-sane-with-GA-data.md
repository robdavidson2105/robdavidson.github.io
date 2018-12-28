---
layout: post
title: "Google Analytics"
subtitle: "How to keep your sanity"
image: interference.jpg
date: 2018-11-19
comments: true
---
# Setting up GA
So you want to understand your customer's behaviour on your website through Google Analytics data?  First of all you need to have your GA data being exported to Google BigQuery - here's the instructions to follow [Setting up BigQuery Export for GA](https://support.google.com/analytics/answer/3416092)

Once your data is going into BQ and your historic back-fill is completed - then you'll begin to realise the scale of the challenge you face to get meaninful insights from the data.

If you have a busy site with lots of customers then your data gets very big, very quickly - and it can be daunting to figure out where to start.

Ok - back to safety ... let's take a look at the data schema first.

The GA export schema is here: [GA Export Schema Definition](https://support.google.com/analytics/answer/3437719?hl=en) - and you can see that for each session/visit you get a whole bunch of nested json objects and arrays.

Hmmm, after this you could still be wondering where to start!

# Explore the data
Start off by getting familiar with the columns available and the data contained in the json objects - use the Google Query console to do this.

### Hits
We're interested in customer behaviour, so obviously the 'hits' column will be useful.   When pulling data from GA we do a a fair bit of 'lateral flattening' ([using Snowflake](https://docs.snowflake.net/manuals/sql-reference/functions/flatten.html)) to pull out things like: hostname, pagepath, pagetitle.

It can also be useful to pull 'timeonsite' from the 'totals' array - to see how much time is spent during each visit session.

Once you have the pages, the times, and the order (use 'hitnumber' for the sequence) - you can build up journeys through your site.

### Devices/Location
As well as behaviour, we're also interested in customer profiles - so exploding the device and geonetwork columns will give us things like the device used, the browser, the screen-size, the operating system version, and an approximate geolocation.

# Aggregate by time interval
Once you've found your interesting data, you're going to need to decide how to feed it into your machine learning models.  For example, if you have thousands of pages on your website, it won't be very useful to one hot encode every page - you'll either need to categorise and aggregate or perhaps use embedding or feature hashing techniques.

How about starting off simple and just define a couple of dozen 'zones' for your website and then aggregate visits to each zone.

Now, you need to look how behaviours alter over time - so you'll probably benefit from pivoting the date-based rows into columns by time interval.

I.e., transform this:

| Date |&nbsp; Fullvisitorid  &nbsp; |   Zone  &nbsp; |   Total Hits |
| --- |:---:| --- |:---:|
| 1 Jan | 0001 | Homepage | 5 |
| 1 Jan | 0002 | Homepage | 6 |
| 1 Jan | 0002 | Basket | 1 |
| 2 Jan | 0001 | Homepage | 2 |
| 2 Jan | 0001 | Basket | 3 |
| 2 Jan | 0001 | Checkout | 5 |
| 2 Jan | 0002 | Homepage | 10 |

Into this:

| Fullvisitorid | Homepage Hits<br>1-Jan | Homepage Hits<br>2-Jan | Basket Hits<br>1-Jan | Basket Hits<br>2-Jan | Checkout Hits<br>1-Jan | Checkout Hits<br>2-Jan |
| --- | ---:|---:|---:|---:|---:|---:|
| 0001 | 5 | 2 | - | 3 | - | 5 |
| 0002 | 6 | 10 | 1 | - | - | - |

You'll need to pivot into time series columns that make sense for you - for example, how about having:
* Total Homepage Hits (current month)
* Total Homepage Hits last month (ie, current month - 1)
* Total Homepage Hits (current month -2) etc...

Now you can see the benefit of aggregating hits by zone - as we pivot by time period, we'll quickly scale-up the number of columns we have as inputs to our model.

# Sanity achieved!
Well - maybe your sanity is still questionable - but at least now you can go ahead and create models that search for patterns of behaviour in GA data.