# Query Project
- In the Query Project, you will get practice with SQL while learning about Google Cloud Platform (GCP) and BiqQuery. You'll answer business-driven questions using public datasets housed in GCP. To give you experience with different ways to use those datasets, you will use the web UI (BiqQuery) and the command-line tools, and work with them in jupyter notebooks.
- We will be using the Bay Area Bike Share Trips Data (https://cloud.google.com/bigquery/public-data/bay-bike-share). 

#### Problem Statement
- You're a data scientist at Ford GoBike (https://www.fordgobike.com/), the company running Bay Area Bikeshare. You are trying to increase ridership, and you want to offer deals through the mobile app to do so. What deals do you offer though? Currently, your company has three options: a flat price for a single one-way trip, a day pass that allows unlimited 30-minute rides for 24 hours and an annual membership. 

- Through this project, you will answer these questions: 
  * What are the 5 most popular trips that you would call "commuter trips"?
  * What are your recommendations for offers (justify based on your findings)?


## Assignment 02: Querying Data with BigQuery

### What is Google Cloud?
- Read: https://cloud.google.com/docs/overview/

### Get Going

- Go to https://cloud.google.com/bigquery/
- Click on "Try it Free"
- It asks for credit card, but you get $300 free and it does not autorenew after the $300 credit is used, so go ahead (OR CHANGE THIS IF SOME SORT OF OTHER ACCESS INFO)
- Now you will see the console screen. This is where you can manage everything for GCP
- Go to the menus on the left and scroll down to BigQuery
- Now go to https://cloud.google.com/bigquery/public-data/bay-bike-share 
- Scroll down to "Go to Bay Area Bike Share Trips Dataset" (This will open a BQ working page.)


### Some initial queries
Paste your SQL query and answer the question in a sentence.

- What's the size of this dataset? (i.e., how many trips)
The dataset is comprised of 983,648 trips.
```sql
#standardSQL
SELECT
  COUNT(*) AS TOTAL_TRIPS
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
```


- What is the earliest start time and latest end time for a trip?
The earliest start time for a trip is 08/29/2013 and the latest end time for a trip is 08/31/2016.
```sql
#standardSQL
SELECT
  MIN(start_date) AS EARLIEST_START_TIME,
  MAX(end_date) AS LATEST_END_TIME
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
```


- How many bikes are there?
There are 700 bikes in the dataset.
```sql
#standardSQL
SELECT
  COUNT(DISTINCT bike_number) AS TOTAL_BIKES
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
```


### Questions of your own
- Make up 3 questions and answer them using the Bay Area Bike Share Trips Data.
- Use the SQL tutorial (https://www.w3schools.com/sql/default.asp) to help you with mechanics.

- Question 1: What are the five most common rides in the dataset and what is the average duration of each? 
  * Answer: The most common ride in the dataset is that from Harry Bridges Plaza to the Embarcadero at Sansome. The average duration of this ride is roughly 1188 seconds, or just south of 20 minutes.
  * SQL query:
```sql
#standardSQL
SELECT
  COUNT(*) AS NUMBER_OF_RIDES,
  start_station_name,
  end_station_name,
  AVG(duration_sec) AS AVERAGE_DURATION
FROM
  `bigquery-public-data.san_francisco.bikeshare_trips`
GROUP BY
  start_station_name,
  end_station_name
ORDER BY
  COUNT(*) DESC
LIMIT
  5
```


- Question 2: What is the geographic range of the bike stations in terms of latitude and longitude?
  * Answer: The bike stations span from 37.2 to 37.8 degrees latitude (roughly 32.8 miles) and from -122.4 to -121.9 degrees longitude.
  * SQL query:
```sql
#standardSQL
SELECT
  MIN(latitude) AS MIN_LAT,
  MAX(latitude) AS MAX_LAT,
  (MAX(latitude) - MIN(latitude))*69 AS LAT_RANGE_IN_MILES,
  MIN(longitude) AS MIN_LONG,
  MAX(longitude) AS MAX_LONG
FROM
  `bigquery-public-data.san_francisco.bikeshare_stations`
```


- Question 3: In which months following the inaugural month of August 2013 were more than one bike station installed?
  * Answer: Three bike stations were installed in December 2013 and two bike stations were installed in each of September 2015, June 2016, and August 2016.
  * SQL query:
```sql
#standardSQL
SELECT
  COUNT(*) AS STATIONS_ADDED_BY_MONTH,
  EXTRACT (MONTH
  FROM
    installation_date) AS INST_MO,
  EXTRACT (YEAR
  FROM
    installation_date) AS INST_YR
FROM
  `bigquery-public-data.san_francisco.bikeshare_stations`
WHERE
  installation_date > '2013-08-30'
GROUP BY
  INST_MO,
  INST_YR
HAVING
  COUNT(*) > 1
ORDER BY
  COUNT(*) DESC
```



