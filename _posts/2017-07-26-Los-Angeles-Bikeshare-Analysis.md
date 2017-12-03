---
category: Blog
---
# LA Bikeshare Data
After working on my [Google Location History Notebook](https://nbviewer.jupyter.org/github/black-tea/google_location_history/blob/master/MyTimeinLA.ipynb), I wanted to dig into spatial data a bit more. I've seen a number of different bikeshare data analyses in the past few years; I thought I would give it a try myself.
### Extract: Get the data from LA Metro
Go to https://bikeshare.metro.net/about/data/ and download all the (1) trip data and (2) station information. As of 7/15/2017, there is one year of trip data released, separated by quarters. Everything is in CSV format.


```python
##### Setup
%matplotlib inline
import urllib.request
import json
import pandas as pd
import geopandas as gpd
from shapely.geometry import Point
import pysal as ps
import matplotlib.pyplot as plt
import datetime
import folium
import xlrd
import glob
import os
from datetime import timedelta
```

##### About LA Metro Bikeshare Data
Attribute information is provided directly on the [LA Metro website](https://bikeshare.metro.net/about/data/). The station information contains the following fields:
* Station ID: Unique identifier for the station
* Station Name: Name of the station
* Go Live Date: Date that the station first became active
* Status: "Active" for stations available or "Inactive" for stations that are not available as of the latest update

The trip data contains the following fields:
* Trip ID ("trip_id"): Unique identifier for the trip
* Duration ("duration"): Duration of the trip, in minutes
* Start Time ("start_time"): Date/Time that the trip began, in ISO 8601 format in local time
* End Time ("end_time"): Date/Time that the trip ended, in ISO 8601 format in local time
* Start Station ("start_station"): Station ID where the trip originated
* Start Latitude ("start_lat"): Y coordinate of the station where the trip originated
* Start Longitude ("start_lon"): X coordinate of the station where the trip originated
* End Station ("end_station"): Station ID where the trip terminated
* End Latitude ("end_lat"): Y coordinate of the station where the trip terminated
* End Longitude ("end_lon"): X coordinate of the station where the trip terminated
* Bike ID ("bike_id"): Unique identifier for the bike
* Plan Duration ("plan_duration"): number of days that the plan the passholder is using entitles them to ride; 0 is used for a single ride plan (Walk-up)
* Trip Type ("trip_route_category"): One Way or Round Trip
* Passholder Type ("passholder_type"): Name of the passholders plan


### Data Cleaning / Formatting
I'll start by creating a few functions to help with the data cleaning and formatting process.


```python
# Load data function to loop through all csvs in a folder and concatentate resulting dfs
def load_data(city):
    path = 'data/' + city + '/trip_data'
    csvs = glob.glob(os.path.join(path,'*.csv'))
    dfs = (pd.read_csv(f) for f in csvs)
    trips = pd.concat(dfs, ignore_index=True)
    return trips

# Function to format trip start/end columns, subset first year of data
def time_format(trips_df):
    trips_df['start_time'] = pd.to_datetime(trips_df['start_time'], errors='raise', infer_datetime_format='True')
    trips_df['end_time'] = pd.to_datetime(trips_df['end_time'], errors='raise', infer_datetime_format='True')
    trips_df = trips_df[(trips_df['start_time'] < min(trips_df['start_time']) + timedelta(days=365))]
    return trips_df

# Load the trip data (4 quarters as of 7/17/2017). 
la_trips = load_data('LosAngeles')
la_trips = time_format(la_trips)

```

### Summary Statistics
Let's start by looking at ridership by month and type of pass. In plotting a chart, I'm going to create a function to store the chart's parameters so that I can quickly create one later again.


```python
def generate_ridership_chart(trips):
    
    # Aggregate by month / year along the count column
    per_month = trips.start_time.dt.to_period("M")
    g = trips.groupby([per_month, 'passholder_type'])
    ridership_by_month = g.size()

    # Create a bar chart showing the percentage of time spent doing each activity
    ridership_chart = ridership_by_month.unstack().plot(kind='bar', stacked=True, title='Number of Trips each Month', figsize=(15,10), fontsize=12)
    ridership_chart.set_xlabel("Year - Month", fontsize=12)
    ridership_chart.set_ylabel("Trips", fontsize=12)
    plt.show()

# Run the function for LA data
generate_ridership_chart(la_trips)
```


![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_6_0.png)


##### Systemwide Ridership by Month
There are about 15k - 20k trips per month for the bikeshare system. As expected, ridership dips down in the so-called winter months and ticks back up inthe spring and summer. However, it is interesting to see the magnitude of ridership dropoff during the winter months, given that the weather in southern california doesn't vary all that much during those months.

The high mark for the program was the second month of operation, August 2016. Why not the first month? The system didn't launch until July 7, 2016, which meant that July was missing a quarter month of ridership. More reasons for the August peak: Metro had an [introductory 50 percent discount rate](http://thesource.metro.net/2016/07/07/its-official-metro-bike-share-program-launches-in-dtla/) that was offered August - September 2016, so walk-up trips that were normally 3.50 were discounted to 1.75; looking at the stacked bar chart above, you can see that both August and September had Walk-Up ridership numbers that have not been attained since. Also, a number of people (including myself) got a free month of ridership; you can see the bump in monthly pass ridership in August that was not present in July or September.

This bar chart seems to show that the monthly pass ridership is surpassing the ridership in the first few months, while the number of walk-ups has declined, and the number of Flex Pass trips has remained more or less the same throughout the entire year. It is interesting that there are so few trips made using the Flex Pass, which is what I currently have. It is a $40 / yr pass that gets you a 1.75 trip 30-minute trip fare, instead of the 3.50 full fare for Walk-ups. Also interesting is the brief period where there were a small number of trips made through a Staff Annual Pass. This program seems to have only existed from October through December of 2016, and doesn't look like it was that popular.


```python
# Aggregate by day of the week
# dt.weekday assigns Monday=0, Sunday=6
per_dow = la_trips.start_time.dt.weekday
ridership_by_weekday = la_trips.groupby(per_dow).size().to_frame('trips')
weekday_labels = [['Monday','Tuesday','Wednesday','Thursday','Friday','Saturday', 'Sunday']]
ridership_by_weekday = ridership_by_weekday.set_index(keys=weekday_labels)

# Bar chart with total weekday ridership
wkday_ridership_chart = ridership_by_weekday.plot(kind='bar',title='Total Number of Trips by Day of Week', legend=False, figsize=(15,10), fontsize=12)
wkday_ridership_chart.set_xlabel('Day of Week', fontsize=12)
wkday_ridership_chart.set_ylabel('Trips',fontsize=12)
plt.show()
```


![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_8_0.png)


##### Bikeshare Ridership by Day of the Week
Looking at the ridership per day of the week, it is interesting, though not surprising, to see Friday as the day with the highest ridership. I believe Friday is the peak day for all trips; it likely also helps that Friday is often more dress casual compared to the rest of the week, so people may be more inclined to ride when they are in more comfortable clothing.


```python
# Subset start_time from trips
trip_times = la_trips[['start_time']]

# Set the trip time as the index, replace the date values to a dummy date, and then delete the column
trip_times.index = trip_times['start_time']
trip_times.index = trip_times.index.map(lambda t: t.replace(year=2000, month=1, day=1))
del trip_times['start_time']

# Bucket the start times in 5 min increments & Plot the chart
trip_times_chart = trip_times.resample('5T').size().plot(legend=False)
trip_times_chart.set_xlabel("Start Time")
trip_times_chart.set_ylabel("Total Trips")
plt.show()
```


![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_10_0.png)


##### Bikeshare Ridership by Time of Day
This is interesting. In addition to the morning and evening peak commute times, there is also a lunch peak period of ridership. This also makes sense given that the stations are located only downtown, so there are fewer people who are able to use Metro Bikeshare to commute compared to those who are downtown for work during the day and want to use the system to get around downtown. 

### How Does Ridership Compare to Other Major Systems?
After 3 months of operation, the [LA Times reported](http://www.latimes.com/local/lanow/la-me-ln-los-angeles-bike-share-ridership-20160919-snap-story.html) that LA's system was not as well used compared to other bikeshare systems throughout the county. The LA Times picked 5 cities on this list (NY, Chicago, SF, DC, Santa Monica) and compared ridership to Metro's system using the metric of trips per bike within the first three months of operation. This is a snapshot of what they found:

![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/la_times_graphic.PNG?raw=true)


Now that we have a full year of data, I wanted to briefly look at comparing the LA system with bikeshare systems in other cities. I could start by looking at those 5 cities in the LA Times study, but I also wanted to expand the comparison to other bikeshare systems. Greater Greater Washington [ranked](https://ggwash.org/view/62137/all-119-us-bikeshare-systems-ranked-by-size) all the bikeshare systems by size, measured by number of bikeshare stations. LA ranks 13th on this list, although I'm sure with the installation of stations in Pasadena (and soon enough in San Pedro as well) I'm sure this ranking will increase. Based on the Greater Greater Washington google spreadsheet, here is a table of the largest bikeshare systems in the U.S.

Rank | City | Stations
--- | --- | ---
1 | New York | 645
2 | Chicago | 581
3 | Washington | 437
4 | Minneapolis | 197
5 | Boston | 184
6 | Miami | 147
7 | Topeka | 138
8 | Philadelphia | 105
9 | Portland | 100
10 | San Diego | 95
11 | Denver | 88
12 | Santa Monica | 86
**13** | **Los Angeles** | **66**
14 | Phoenix | 63
15 | Buffalo | 63


From this table, you can see that there were several systems larger than Metro's but did not make the LA Times study, including Minneapolis, Boston, Miami, Topeka, Philadelphia, Portland, San Diego, and Denver. I want to include some of these in the analysis here.

 

### Generate Charts for Other Bikeshare City Data

Here I load in and reformat the data from other bikeshare systems. Since most provide the files by calendar year, I load in two years of data and then filter out anything that is more than 1 year after the first ride in the system.

Some notes from looking at the initial one-year data of other systems-

* Just by looking at the way the data is formatted, you can immediately tell which systems share the same operator. Philadelphia and Los Angeles, for example, are operated by Bicycle Transit Systems, while 
* A number of systems go dark in the winter.
* A few of the systems don't include a bicycle_id field, which precludes me from doing the more detailed trips/bike/day calculation that I performed for LA.




```python
# Load data function to loop through all csvs in a folder and concatentate resulting dfs
def load_data(city):
    path = 'data/' + city + '/trip_data'
    csvs = glob.glob(os.path.join(path,'*.csv'))
    dfs = (pd.read_csv(f) for f in csvs)
    trips = pd.concat(dfs, ignore_index=True)
    return trips

# Function to format trip start/end columns, subset first year of data
def time_format(trips_df):
    trips_df['start_time'] = pd.to_datetime(trips_df['start_time'], errors='raise', infer_datetime_format='True')
    trips_df['end_time'] = pd.to_datetime(trips_df['end_time'], errors='raise', infer_datetime_format='True')
    trips_df = trips_df[(trips_df['start_time'] < min(trips_df['start_time']) + timedelta(days=365))]
    return trips_df

# Load / Format Denver Data
denver_path = 'data/Denver/trip_data/2010denverbcycletripdata_public.xlsx'
dv_trips = pd.read_excel(denver_path, parse_dates={'start_time':['Check Out Date', 'Check Out Time'],
                                                   'end_time':['Return Date','Return Time']})
dv_trips.rename(columns = {'Membership Type':'passholder_type',
                           'Bike':'bike_id',},
               inplace=True)

# Load / Format Minneapolis Data
mn_trips = load_data('Minneapolis')
mn_trips.rename(columns = {'Start date': 'start_time',
                           'End date':'end_time',
                           'Account type':'passholder_type',
                           'Start terminal':'start_station_id',
                           'End terminal':'end_station_id'},
                inplace=True)
mn_trips = time_format(mn_trips)

# Load / Format Boston Data
bs_trips = load_data('Boston')
bs_trips.rename(columns = {'Start date': 'start_time',
                           'End date':'end_time',
                           'Member type':'passholder_type',
                           'Bike number':'bike_id',
                           'Start station number':'start_station_id',
                           'End station number':'end_station_id'},
                inplace=True)
bs_trips = time_format(bs_trips)

# Load / Format DC Data
dc_trips = load_data('WashingtonDC')
dc_trips.rename(columns = {'Start date': 'start_time',
                           'End date':'end_time',
                           'Member Type':'passholder_type',
                           'Bike#':'bike_id'},
                inplace=True)
dc_trips = time_format(dc_trips)

# Load / Format Philadelphia Data
pa_trips = load_data('Philadelphia')
pa_trips = time_format(pa_trips)

```    
    


```python
print(min(mn_trips['start_time']))
```

    2010-06-07 16:42:00
    


```python
# Generate the ridership charts
generate_ridership_chart(dv_trips)
generate_ridership_chart(mn_trips)
generate_ridership_chart(bs_trips)
generate_ridership_chart(dc_trips)
generate_ridership_chart(pa_trips)
```


![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_16_0.png)



![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_16_1.png)



![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_16_2.png)



![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_16_3.png)



![](/images/2017-07-26-Los-Angeles-Bikeshare-Analysis/output_16_4.png)


### Station Information

#### Where is Metro Bikeshare?
I wanted to start by looking at the stations on a map. However, since the station information table doesn't contain lat/lon coordinates, I would need to merge the tale with the trip data OR use the [updated GeoJSON feed](https://bikeshare.metro.net/stations/json/) on the LA Metro website. I opted for the latter. 

#### Bikeshare Ridership by Station
Let's look at the differences in ridership by station, separating origins and destinations.


```python
# Initially getting HTTP Forbidden Error, so added headers
url = "https://bikeshare.metro.net/stations/json/"
hdr = {'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.11 (KHTML, like Gecko) Chrome/23.0.1271.64 Safari/537.11',
       'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
       'Accept-Charset': 'ISO-8859-1,utf-8;q=0.7,*;q=0.3',
       'Accept-Encoding': 'none',
       'Accept-Language': 'en-US,en;q=0.8',
       'Connection': 'keep-alive'}

# Load the GeoJSON
req = urllib.request.Request(url, headers=hdr)
try:
    response = urllib.request.urlopen(req)
except urllib.request.HTTPError as e:
    print(e.fp.read())
station_json = json.loads(response.read())

# Import to GeoDataFrame
stations = gpd.GeoDataFrame.from_features(station_json['features'])

# Create basemap and add station points
station_map = folium.Map([34.047677, -118.3073917], tiles='CartoDB positron', zoom_start=11)
for index, row in stations.iterrows():
    folium.Marker([row.geometry.centroid.y, row.geometry.centroid.x]).add_to(station_map)
station_map
```
    




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2FqYXguZ29vZ2xlYXBpcy5jb20vYWpheC9saWJzL2pxdWVyeS8xLjExLjEvanF1ZXJ5Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1IiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGJvdW5kcyA9IG51bGw7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgdmFyIG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSA9IEwubWFwKAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgJ21hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NScsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7Y2VudGVyOiBbMzQuMDQ3Njc3LC0xMTguMzA3MzkxN10sCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB6b29tOiAxMSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIG1heEJvdW5kczogYm91bmRzLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgbGF5ZXJzOiBbXSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHdvcmxkQ29weUp1bXA6IGZhbHNlLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgY3JzOiBMLkNSUy5FUFNHMzg1NwogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHRpbGVfbGF5ZXJfNzZiM2Y0MGZmNjFlNDE4NmIxNWZiYTFkNjI5YTRjZGYgPSBMLnRpbGVMYXllcigKICAgICAgICAgICAgICAgICdodHRwczovL2NhcnRvZGItYmFzZW1hcHMte3N9Lmdsb2JhbC5zc2wuZmFzdGx5Lm5ldC9saWdodF9hbGwve3p9L3t4fS97eX0ucG5nJywKICAgICAgICAgICAgICAgIHsKICAiYXR0cmlidXRpb24iOiBudWxsLAogICJkZXRlY3RSZXRpbmEiOiBmYWxzZSwKICAibWF4Wm9vbSI6IDE4LAogICJtaW5ab29tIjogMSwKICAibm9XcmFwIjogZmFsc2UsCiAgInN1YmRvbWFpbnMiOiAiYWJjIgp9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjIzOGRkNjY1NzI1NGM3NGJkM2ViMDE5OWU1MTU5NjQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDg1LC0xMTguMjU4NTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzYyMDM0ZWMyNWVmZjRiYzk5NmFkNGI0NjBlNWIwNGNiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQ1NTQsLTExOC4yNTY2N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNzRmYjVhNTEzMWY5NGFlZTlmMjZiNTMwZWMxOGQwMDYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTA0OCwtMTE4LjI1NDU5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kNTRkYTFmY2ZiMWU0OGQxOTFjY2ZjMTM4YmVkODgxYSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0NjYxLC0xMTguMjYyNzNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2Q2ZWZkZmM4ZTdiNzRkMjlhZGJlYTJhMTg2NGMwNWQ0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM3MDUsLTExOC4yNTQ4N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzFjOTcyMTU0YzVlNGU4NjgyYzVmNDI0OTY2ZjY2ODggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDExMywtMTE4LjI2Nzk4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xOTc5M2ZlZTE2NzU0ZDVlYWRjYzg0YzY2YTI4YTdjNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjc3OTgyLC0xMTguMjYzMDJdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M0OWZlMWQ3ZWYzMzQwM2RhNDY4MTlmMjkxNDAwZjJjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDU2NjEsLTExOC4yMzcyMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMDFmMjE0YzM4MjYyNGM5Zjk4MmZhMmZmM2QxOTk5NmEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTI5LC0xMTguMjQxNTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ZiYmMwYjFmMDk2ODQ1YWU5MGU1OWI3YWFjOGE1MTJiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQzNzMsLTExOC4yNjAxNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTEzOTZkY2U3ZThhNDYwZGFiNjRkNTljZmU0MzE5NDQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wMzg2MSwtMTE4LjI2MDg2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wZTlkMDAwYjUyYzM0Mjc1YWY3YjcwNWU4OWZkYWIzYiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjAzMTA1LC0xMTguMjY3MDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzBjZTdhYzk4M2QzMjQwN2RhZjFiOTY3ZDg3ZDQ3MmFkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQ2MDcsLTExOC4yMzMwOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYWVmMjIwOWJlNzkwNGM0ZWI2Nzk1MDcyYWQ1ZWNjZGMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTA5MSwtMTE4LjI0MDk3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9mZTgyMTA3YWM4M2Q0MTVmODRlZGZhMzg2MTY0MTk1MyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1NzcyLC0xMTguMjQ4OTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzYyMGNiZmRjZDk3ZTQ2ZjE4MjgxOTFmNDk2YzRjMjFkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDMyODYsLTExOC4yNjgwOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjUxOWFmNzZjOTQzNGI1ZDliNzUxYzc3YzE2N2U0YjIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNjMxOCwtMTE4LjI0NTg4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kOGE5NjY5YWY5YTg0YWVmODljMDRjMjM5M2IxN2ExOSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0OTk4LC0xMTguMjQ3MTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2FhYTA0NGRmM2RkMDQ1NmE5YmZjNmFmYWIwMTQ0MTUxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDU4MzIsLTExOC4yNDYwOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTk3YzJkMzgwYzNiNDNhYmJjYTYwMWVjOTM0YTBkMTcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDg4NSwtMTE4LjI0NjQyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lNDc2NzNlN2MzOWQ0NWIyYTI3OTBhZmUyMDk3NTgyZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1MTk0LC0xMTguMjQzNTNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQwM2E1OTQ0MGE0MjQ1Y2I4MThiMTc5MjVhMjNlZTkzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQ0NywtMTE4LjI1MjQ0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9hOGZkZGEzOTFhYWI0YTQ0ODlhMmI3OTIwNDA0NTQ3ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0OTg5LC0xMTguMjU1ODhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzIwMGVjNWEyNWU3NTQ0YWNiOTRhNTI5M2I0MWE4YzQyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQwOTksLTExOC4yNTU4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kMzJjMzU0NThiZjU0NTM1OThhNDYxMDViMGE0NzI0MSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0MjA2LC0xMTguMjYzMzhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2VkY2QwZTM3MTZjYTQwMTY4YjNmZGUxNzE3Y2E1YWZlID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQ4NCwtMTE4LjI2MDk1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iNzEwZmViZDBkNzY0N2NjYTAxY2RlOWRlY2Y4Yjg2OSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjAzOTE5LC0xMTguMjMyNTNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzNjMGQyNjgwOGY3NjRkMDA4ZDRmYTYxN2VlMTA4ZmMwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM0OCwtMTE4LjIzMTI4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wYzZiY2M2ODkzMzM0NzIwOTAwNmY0OGI0OTY4NzMyMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0NjgyLC0xMTguMjQ4MzVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzJhYjBkNjhlMWE3ODRjYmU4ZTViN2JkNWVlMWJiZjdhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDUzNTcsLTExOC4yNjYzNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNmJkYWJmYmFiZDdjNGUxYWFkZDdjZTcyZjk3ZDAxOTAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDkzLC0xMTguMjM4ODFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzMwN2Y3ZWVlMTk1NjQ5YTY4ODIwZGJmYzk5M2E1OTQxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDI4NTEsLTExOC4yNTY2N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMjA2Y2U2ZGQ2ZTgzNDM0OGFiZGUzOGM3NTQ1NmI5MTAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTMwMiwtMTE4LjI0Nzk1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85NTY4NGUyMGJlNDQ0OTZjOTQyMDY0NWU2ZTVkYTYzYSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjAzOTk4LC0xMTguMjY2NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNjdlYjQzNjk2ZjUyNDJmNTg3MTI3OGI1NmFkODQ4ZGYgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDE2OSwtMTE4LjIzNTM1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8wMzFkOTM4NDAwODM0MGIwOWQ2YjE1NTdkYmI3YjZmYiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1Njk3LC0xMTguMjUzNTldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQ2NTY1YWVmODYwNzQzM2ZhODY3NWFiYTAyNjU5ODQ2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQ1NDIsLTExOC4yNTM1Ml0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzBjNDEzMWMzZTEzNGEyMzhjYWM2NmMyMjJiYzQxYWUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTExLC0xMTguMjY0NTZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2U1MDUxNGQ0ZTFhZTQzZDM4YzFlYTUzZDQ5ZjZjYjhmID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM5MjIsLTExOC4yMzY0OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGY5MmZhZDlkOTA0NGFlMTg2ZjBlMDY0MmJlYzI0YzkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDQxNiwtMTE4LjI1MTU4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80NmJhMWI2NDRhNTg0NmQ2OWE2MjU5MDQ5NWJlZjg4NyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjAzNzQ2LC0xMTguMjY1MzhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzcxMzBiODdiNjk5MzQyOTk5ZjVlMzk4NTBmNjhkYTA0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM1NjgsLTExOC4yNzA4MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYWNiNGUzNDNhYzc4NDcwODlkOTk5YWU1ZTI1M2FkZTggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wMzU4LC0xMTguMjMzMTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzgwOTM3NDhlODVhNTQ4YmRhYTVmMWRhYTg5ZTQ2ZjE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM0ODgsLTExOC4yNTc2OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZDBlYjFiY2VkNDk5NDNlNzljZWQ0NmJkYWE4MjExYzkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDc3NSwtMTE4LjI0MzE3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8yY2YxMWY4MDFkZTA0YWI2Yjc1YjRjYzA0NWI3ZWM2NiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0OTIsLTExOC4yNTI4M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMGYyYjhhZjI4NjBmNDhmYmIwMDhmNGQxNjNjZmFmZjggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDY4MSwtMTE4LjI1Njk4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kMDRlYTU3N2FjZTU0ZDM0YjBhNmY0YjEwMDM0NmE5YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA2MDU2LC0xMTguMjM4MzNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzg5Y2JjZjdjNDI0ZjRkYjk4MDY0MjIzYjQwODlmMzE2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDYzMzksLTExOC4yMzYxNl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYTRkZjA5YTQ2NWJmNDQ2Y2E1MGQ2YmRhOTYwMGJiZmIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDUxOCwtMTE4LjI1MDI0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9mNGEwN2ZlNTYzMWY0MWUyYjdkYjdhZWI3MTFjN2VhMCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1MzIsLTExOC4yNTA5NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYmU5YzY0NDc2M2Q1NGJjOGI1M2U0MDQzYWJhNzU4YjUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNTA4OCwtMTE4LjI0ODI1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iNDlmOWM0MTUyMDk0NDc4YWI2YTMxMTg1YWUxODY0MCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0NDE3LC0xMTguMjYxMTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzM0NjJjNmY4ZjcwNTQwMmVhOWRiZWFjOGU5OWI3MzdhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDQyMTEsLTExOC4yNTYxOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTZmYjJhMWQ5ZjQ1NDNiYjg1MGJhYjc5Y2I4NzBlZDMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNDA2LC0xMTguMjUzODRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2JlMmNjZDJlYzdlZjQ3ODM4YmVkNDhlZTVlODFkZGNiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM5ODcsLTExOC4yNTAwNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNmY0NmVkMjY3OTU5NGM4YzhlOGExYTY0NjBlNjNhODcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNjQyOCwtMTE4LjIzODk0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iZWExNWY4ODk0OTM0YzA0YjNmYmU5ZjA3MjU1ODJkYyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA1MDE0LC0xMTguMjMzMjRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzFkMzFkNzYyODM0NzQ4ZTNhOTRjNTUzY2RlMjBmYTEwID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDM0MjEsLTExOC4yNTQ1OV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDZlYzNhZTM1YjgwNGU0ZDkxZGM0Yjk5N2Q3NzQ5ODIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wMzE4OSwtMTE4LjI1MDE4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80OWFkYTM3NmFiZjk0ZjE4YjA5NGRiNTZjZmY2ZDMxNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjA0NjUyLC0xMTguMjM3NDFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQwMTBiMGY0ZTUxNjRjOGJhY2ZiY2UwNjMzMDM4MzkxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuNzY2NjYsLTExOC4yNjEwMl0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfY2I4YzdiZGI4YTdjNDk4MDhjNjcwNzFmMjZhYTJmM2UgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy43MTA5OCwtMTE4LjI4NDY3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZWExNmM2YWEzOGQ0Mzg5YjVmMmI2N2QyNDdlNDQ0YSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjc0ODkyLC0xMTguMjc1MTldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQ4OGUxMzRkZDk4YTQzZTI4MzhlZGFmMTc4MzNhMmI2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuNzI1ODIsLTExOC4yODA1MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOGI2OWJkMmU1MWFmNGU4N2IwMTBhNTc0NzViYTkyZGEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy43MTkyNiwtMTE4LjI4MjcyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84ODU4MzNkZDkyYTI0NmJmYTUzYTk5MTk3Mzc0OWMyMyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjczODk4LC0xMTguMjc5MjddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzI4ZjEwOTFjZmUyYjRmYzFiN2NmMDI1OGZjNzRjZjFjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuNzQ5MTMsLTExOC4yODAxOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMTllMmYwZDVlNmRhNDMyZWExNGYxNjM0ODg0Nzc4YWEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy43MzM1MiwtMTE4LjI3Nzc5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kYzNjN2I0MDVmYWQ0NzgzOGMyYTY3MTk3MDJkZTM5ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjc0MTM2LC0xMTguMjc4MDddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzBhMWM1MDFiNjliNDQ4YzJiYWU2YTMwODg1MDE4MTNhID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuNzcyMywtMTE4LjI2ODA0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8zMGVjYjVlYjEwNTY0YzQ5ODAyZWY4NzZlNjg4ODA0YiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjc3MTc2LC0xMTguMjc2NTRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2M4YmFmMDdlMGNmMDQwZjNhODMyMzMzZWY2YjU1MDUzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTQ1NjksLTExOC4xNDgyNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNDM4MTFjN2VkMGZhNDgzNGFhMjRlMjc3MjMxODNmMjQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDQ1OSwtMTE4LjE0NDU5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80N2I4Mzg2ODFlMGY0MmFkYTc2N2ZlOWY5ZWZkYzdjZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0MjQsLTExOC4xMzI2NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZjRjM2ZlMDkwNjViNDZhYzg0YzIyMWJhMjgxOGFmNmEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDc4MywtMTE4LjE0NTA0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iNTc3ZWUxMmZiNjY0OGFiYTM2NjdiN2JiMmJkNGVhNSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0NTI1LC0xMTguMTUwMDddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I1YmE1NzVmMDQ0NjQ3MTg4YmQyY2VmZmU2ZDEyM2RjID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTQ5NjcsLTExOC4xNDUxMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTYyN2UzZGNmOGRkNDAwMDgyZDRlNDQzMjVlMTU3MDkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xMzgyMSwtMTE4LjE0NzA4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl80MzA5ZmQwNDk2NGQ0ZDg2YjY0ZmVjNTI0OTNkNTc4ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE1NDI3LC0xMTguMTQyMTVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzQ0MDY2MjlmMTdjODRkM2M4YjdmYjU5YTg3OWQ1MjNkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTQyMzYsLTExOC4xNDE1MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZmRjYzJkYmUzYmQ1NGI3NjhlNzU4ZTFjN2FlMTliMTkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDk3MSwtMTE4LjE2MDE3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xYjMyNTFmNzA0YzM0M2U1ODk0ZTEwODZkYTBhZGQ3ZSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE1NjE4LC0xMTguMTY2NjldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2MzMTRkYzEyMjBkMzQwZGQ4NGU3MTc0YWE2ZWNjYjhkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTU2MTIsLTExOC4xNTExOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOTEyNGVkNTNmM2ZiNGI1NmI1YmZhMmFjOWM4ZTE5NzkgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDc4MSwtMTE4LjEzMjIyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl83YzcxNTk4MzlmNmE0OThhOWFmNDI1YjEzNWNiN2NmOCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0OTkxLC0xMTguMTM4MzldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I4N2RjZDhiZjc4MTQ3NjVhNDlmN2JhZmNjYTkwNjY2ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTQxLC0xMTguMTMyMDldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzExZTNiY2UwYmE5NzQ0NzliZTU2NTBkNmVlNDcyODI0ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTM1MjUsLTExOC4xMzIzN10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYzg2NjJmZDQ1NzllNGJmZWE5ZTA1YzFjMGQzNTM2NzMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xMzMwMSwtMTE4LjE0ODU2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85MTQ0MjcxYzEwOWU0OGRkYjA4N2YwZGFiYWYxODkxMSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0NzUsLTExOC4xNDgwMV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfY2QxNzVlNTQ2MTE2NGY2OGJkNTUyMTM0ZjgxYjQ4MTcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNTA0NywtMTE4LjEzMjAyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84YjEyNWU2YzIyZWE0YjA4OGY3YTJkODcwNzQ3MjA4YyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0MTc1LC0xMTguMTQ5MDZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2U1MDkwNDI0ODY5YjRjN2M5NDBmYjAzNmQyOTM2ZTcxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTQzODIsLTExOC4xNTQxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lNmMxMGUzYjkxZjY0ODEyYjUyNDcxNTI5YTlkMjM5OSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE0Njk2LC0xMTguMTM5ODRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzk0NzE4MjI1NWUxMTRiYjk4Y2ZlNjU0OTdiMzk4YmE4ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTYwNzEsLTExOC4xMzI0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl82MDU0ZTc1OGU1ZWE0ZDZmOGFiZjY3NzAyYTUzMTVhMiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjE2NTI5LC0xMTguMTUwOTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2EyYzU2OGFiYzMwMTQzMDI5YTljMTE4OTk5ZWIzYTAzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMTM3ODYsLTExOC4xMjI0MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNmY3YzEwOGM5ODJhNDIxZjgwYzk1OTZhZTQyMzgxNzEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDQ5OCwtMTE4LjEzODI2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9lMjAxZjE0ZTQyNmQ0NjlmYjJlNjg1YTFmMzM5ZmY1MiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjEzNzk1LC0xMTguMTI4NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZjY4ODNhYTU4ZDc1NDBiNzhhZDgxOTVlOWRhM2Q5NmIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4xNDYyMiwtMTE4LjEzNTI2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9kN2QxMGMxMTYwNmM0NzYwOTY2NDA2OTcwMzk5M2U3ZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjk5MTE2LC0xMTguNDY4MjldLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzIzMGZjNzc3YjMxZDRjMTRiMzE0ZjgyZmMyMmNjMTY5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuOTg4NDIsLTExOC40NTE2M10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYTRiYTljZDBhNTQxNDFhZDkzZDQxMTFmYmVmN2MyNzggPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy45OTM3NywtMTE4LjQ1MzY3XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl84ZjlhMmE5YmNmMDY0MmY4OGJjYmIwMGJmYWEzZWY2MSA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjk5ODM0LC0xMTguNDYxMDFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2YwOTQyMmI1MTJjZTQxYTc5MTNiMWVlNGQ1MGQyNGVmID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDAwODgsLTExOC40Njg5MV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfYmRmMzgxNTliZjM0NDA4YTg4ZWYyNDFlNmI5ODVjZjcgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy45OTg2OCwtMTE4LjQ3Mjk4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8yYmYyM2NhNTk5OTI0ZjA5ODc4YTRjNDllMDc0OThkNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjk5NjI0LC0xMTguNDc3NDVdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2Q3Zjk2ZGQyYTU1YzRlYTc5NWQ1MjZlODZmZDUxZWRkID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzMuOTg0MzQsLTExOC40NzE1NV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfOWI1MTIzNzI0NmY4NDM2OTkyYjNmZTY0MzU1ZWQ4NDIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy45ODQ5MywtMTE4LjQ2OTk2XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9iZjc0MWMyMWU2YzI0YTRkOGQyM2IzYmRjYWJjMmNmNCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzMzLjk5NDcsLTExOC40NjQwOF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZmJjN2I5ODlkMzExNGRkNmIzYmVmYmJmNmMyYjZkZDAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszMy45OTU1NiwtMTE4LjQ4MTU1XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xNjkyMDYxODQ1M2Q0OTFjOWE4ZTFjNmYxNzY4OTcwMCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzM0LjAxNDMxLC0xMTguNDkxMzRdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF9lNTNkOTRiZDI1MGU0MmE3YTc4NDdhYzI1ZDMyOTg0NSk7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzBhYTgzYzk1M2JkZjRmMTBiNjU2ODFkNWZlYWE3MTJiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbMzQuMDIzMzksLTExOC40Nzk2NF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2U1M2Q5NGJkMjUwZTQyYTdhNzg0N2FjMjVkMzI5ODQ1KTsKICAgICAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZjRkNzM4MDYxMGRiNDM4MzhlOTJhYWM5MmNlYTUxZDMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFszNC4wNzQ4MywtMTE4LjI1ODczXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfZTUzZDk0YmQyNTBlNDJhN2E3ODQ3YWMyNWQzMjk4NDUpOwogICAgICAgICAgICAKPC9zY3JpcHQ+" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>




```python
# Load the station file
station_path = 'data/LosAngeles/station_data/metro_station_table.csv'
stations = pd.read_csv(station_path)

# First reformat stations df by setting station id as the index
stations = stations.set_index('Station ID')

# Count origins and destinations
stations_o_count = la_trips.groupby(la_trips['start_station_id']).size().to_frame('o_ct')
stations_d_count = la_trips.groupby(la_trips['end_station_id']).size().to_frame('d_ct')

# Combine O/D counts with station information
stations_od_count = pd.merge(stations_o_count, stations_d_count, right_index=True, left_index=True)
stations_count = pd.merge(stations, stations_od_count, left_index=True, right_index=True)
stations_count.head()

# Print out the max / min trip stats
print("The station with the highest number of origin trips is{}".format(stations_count.loc[stations_count['o_ct'].idxmax()][0]))
print("The station with the highest number of destination trips is{}".format(stations_count.loc[stations_count['d_ct'].idxmax()][0]))
```

    The station with the highest number of origin trips is Broadway & 3rd             
    The station with the highest number of destination trips is Flower & 7th               
    
