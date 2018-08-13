# template-activity-03


# Query Project

- In the Query Project, you will get practice with SQL while learning about
  Google Cloud Platform (GCP) and BiqQuery. You'll answer business-driven
  questions using public datasets housed in GCP. To give you experience with
  different ways to use those datasets, you will use the web UI (BiqQuery) and
  the command-line tools, and work with them in jupyter notebooks.

- We will be using the Bay Area Bike Share Trips Data
  (https://cloud.google.com/bigquery/public-data/bay-bike-share). 

#### Problem Statement
- You're a data scientist at Ford GoBike (https://www.fordgobike.com/), the
  company running Bay Area Bikeshare. You are trying to increase ridership, and
  you want to offer deals through the mobile app to do so. What deals do you
  offer though? Currently, your company has three options: a flat price for a
  single one-way trip, a day pass that allows unlimited 30-minute rides for 24
  hours and an annual membership. 

- Through this project, you will answer these questions: 
  * What are the 5 most popular trips that you would call "commuter trips"?
  * What are your recommendations for offers (justify based on your findings)?


## Assignment 03 - Querying data from the BigQuery CLI - set up 

### What is Google Cloud SDK?
- Read: https://cloud.google.com/sdk/docs/overview

- If you want to go further, https://cloud.google.com/sdk/docs/concepts has
  lots of good stuff.

### Get Going

- Install Google Cloud SDK: https://cloud.google.com/sdk/docs/

- Try BQ from the command line:

  * General query structure

    ```
    bq query --use_legacy_sql=false '
        SELECT count(*)
        FROM
           `bigquery-public-data.san_francisco.bikeshare_trips`'
    ```

### Queries

1. Rerun last week's queries using bq command line tool (Paste your bq
   queries):

- What's the size of this dataset? (i.e., how many trips)
```sql
  bq query --use_legacy_sql=false '
SELECT
  COUNT(*) AS TOTAL_TRIPS
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips` '
```

- What is the earliest start time and latest end time for a trip?
```sql
  bq query --use_legacy_sql=false '
SELECT
  MIN(start_date) AS EARLIEST_START_TIME,
  MAX(end_date) AS LATEST_END_TIME
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips` '
```

- How many bikes are there?
```sql
  bq query --use_legacy_sql=false '
SELECT
  COUNT(DISTINCT bike_number) AS TOTAL_BIKES
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips` '
```

2. New Query (Paste your SQL query and answer the question in a sentence):

- How many trips are in the morning vs in the afternoon?
```sql
  bq query --use_legacy_sql=false '
SELECT
  COUNT(*) AS RIDES,
  CASE
    WHEN EXTRACT(HOUR  FROM  START_DATE) BETWEEN 6 AND 12 THEN "MORNING"
    WHEN EXTRACT(HOUR  FROM  START_DATE) BETWEEN 12 AND 18 THEN "AFTERNOON"
    ELSE "NIGHT"
  END AS TIME_WINDOW
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY
  TIME_WINDOW'
```
Answer: There are 428,818 trips during the morning (defined as starting between 6am and noon) versus 446,771 trips during the evening (starting between noon and 6pm).


### Project Questions
Identify the main questions you'll need to answer to make recommendations (list
below, add as many questions as you need).

- Question 1: What percentage of rides are taken by subscribers? What percentage are taken by 24-hour or 3-day members? 

- Question 2: Does the average trip duration vary based on rider type?

- Question 3: What are the most common trips grouped by time window?

- Question 4: Which stations have the highest dock utilization rates? This would be a good place to start if/when we are looking to add docks to a station. (Also a note that we are assuming consistent station logging here.)

- ...

- Question n: 

### Answers

Answer at least 4 of the questions you identified above You can use either
BigQuery or the bq command line tool.  Paste your questions, queries and
answers below.

- Question 1: What percentage of rides are taken by subscribers? What percentage are taken by 24-hour or 3-day members?
  * Answer: 86% of rides are taken by subscribers and 14% of rides are taken by 24-hour or 3-day members.
  * SQL query:
```sql
  bq query --use_legacy_sql=false '
SELECT
  SUBSCRIBER_TYPE,
  COUNT(*) AS RIDES,
  COUNT(*) / (
  SELECT
    COUNT(*)
  FROM
    `bigquery-public-data.san_francisco.bikeshare_trips`) AS PERC_RIDER_TYPE
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY
  SUBSCRIBER_TYPE'
```

- Question 2: Does the average trip duration vary based on rider type?
  * Answer: Yes, there is a staunch difference in average trip duration based on rider type: the average customer trip lasts 3718 seconds (just over an hour) while the average subscriber trip lasts 583 seconds (just under ten minutes). This makes sense intuitively as customers are likely to try “getting their money’s worth” out of any given transaction.
  * SQL query:
```sql
bq query --use_legacy_sql=false '
SELECT
  SUBSCRIBER_TYPE,
  COUNT(*) AS RIDES,
  COUNT(*) / (
  SELECT
    COUNT(*)
  FROM
    `bigquery-public-data.san_francisco.bikeshare_trips`) AS PERC_RIDER_TYPE,
  AVG(duration_sec) AS AVG_TRIP_DURATION
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY
  SUBSCRIBER_TYPE'
```


- Question 3: What are the most common trips grouped by time window?
  * Answer: The two most common trips are 2nd at Townsend to Harry Bridges Plaza during the afternoon and the inverse during the morning. This is a good example of a probable commuter trip.
  * SQL query:
```sql
bq query --use_legacy_sql=false '
SELECT
  COUNT(*) AS RIDES, START_STATION_NAME, END_STATION_NAME,
  CASE
    WHEN EXTRACT(HOUR  FROM  START_DATE) BETWEEN 6 AND 12 THEN "MORNING"
    WHEN EXTRACT(HOUR  FROM  START_DATE) BETWEEN 12 AND 18 THEN "AFTERNOON"
    ELSE "NIGHT"
  END AS TIME_WINDOW
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY
  TIME_WINDOW, START_STATION_NAME, END_STATION_NAME
ORDER BY COUNT(*) DESC
LIMIT 10'
```


- Question 4: Which stations have the highest dock utilization rates? This would be a good place to start if/when we are looking to add docks to a station. (Also a note that we are assuming consistent station logging here.)
  *Answer: Stations 50 & 70 have higher than 58% utilization rates.
  *SQL Query:
```sql
SQL Query:
  bq query --use_legacy_sql=false '
SELECT
  COUNT(*) AS LOGS,
  STATION_ID,
  AVG(bikes_available) AS AVG_BIKES_AVAIL,
  AVG(docks_available) AS AVG_DOCKS_AVAIL,
  AVG(bikes_available) / (AVG(bikes_available) + AVG(docks_available)) AS PERC_UTILIZATION
FROM
  `bigquery-public-data.san_francisco.bikeshare_status`
WHERE
  bikes_available + docks_available > 0
GROUP BY
  station_id
ORDER BY
  PERC_UTILIZATION DESC
'
```

- ...

- Question n:
  * Answer:
  * SQL query:
