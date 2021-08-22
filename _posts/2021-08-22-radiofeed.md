---
title: "Radio station feed data analysis"
date: 2021-08-22
tags: [data wrangling, data analysis, data visualizations, python]
header:
  image: "/images/radiofeed/radioclash.jpg"
excerpt: "Data Wrangling, Data Analysis, Data Visualizations, Python"
mathjax: "true"
---

# Radio station feed data

## Context
Everytime a song is played on the radio, its airplay is reported to a performing rights organization. There are many such organizations, depending on the country/continent the radio station is based on. One common issue with the logging of the airtime is in the data entry itself. Duplicate entries, erroneous song naming, missing names in general etc. Therefore, data cleaning is even more important when dealing with radio play data.

## Introduction
This notebook deals with song play data from various radio feeds. They span a period of 21 months from January 2019 to September 2020. Using the supplied datasets, songs can be pinned to their respective feeds and feeds to their respective radio stations. The focus of this notebook will be on the following:

- Cleaning the dataset
- Visualization and insights 

The data cleaning will focus on identifying and removing duplicate entries. Duplicate entries will have an impact in the revenue stream towards the artists.

The visualizations in this notebook will be done using Plotly which offers out-of-the-box chart interaction. The cleaned dataset will be also fed into Tableau for further visual representation of the data.

To use Plotly in JupyterLab, install the jupyterlab and ipywidgets packages using pip:

$ pip install "jupyterlab>=3" "ipywidgets>=7.6"

or conda:

conda install "jupyterlab>=3" "ipywidgets>=7.6" 

For Plotly in Jupyter Notebook, install the notebook and ipywidgets packages using pip:

$ pip install "notebook>=5.3" "ipywidgets>=7.5"

or conda:

$ conda install "notebook>=5.3" "ipywidgets>=7.5" 


## Data wrangling

First let's load the datasets and the packages we'll need.


```python
# Preamble
import pandas as pd
import numpy as np

import matplotlib.pyplot as plt
from matplotlib import style
import seaborn as sns

import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots
from ipywidgets import widgets

# Load and preprocess the data

## Data consumption
cd_df = pd.read_csv (r'consumption_data.csv')

## Feed station mapping
fsm_df = pd.read_csv (r'feed_station_mapping.csv')
```

Before the datasets merge, it's good to examine each separately to easier locate errors. Pandas *.info()* and *.describe()* give an overview of the dataset. Let's take a look at the consumption data first. 


```python
cd_df.info()
cd_df.head(5)
cd_df.describe()
```

A few things stick out from the dataframe profile:

- Timestamp has 137662 duplicate values. This can mean possible duplicate plays for same time, feed and radio station.
- Discrepancy in feed_ids and feed_names. This will be addressed a few lines below, once the feed station mapping data is examined as well.
- Only 8 songs and 8 artists played over 21 months in 1000+ stations. This is can hint that the data are focused to a specific artist/song catalogue (maybe a certain record label?).

In addition there is an error in the track column name. Listing the columns of the consumption data datafrane, the error comes from a very annoying (and very common) leftover space, visible with *list(cd_df.columns)*. 

This is easily fixed with a simple column rename.


```python
print('Before:', list(cd_df.columns))
cd_df.rename(columns = {'Track ':'Track'}, inplace = True)
print('After:',list(cd_df.columns))
```

The duplicate rows in the consumption data are 4909. Consider these as data errors at the reporting level, since a song (defined as a unique combination of 'Track' and 'Artist') cannot be played more than once, at a specific timestamp, on the same station.


```python
dupes_cd = cd_df.duplicated().sum()
cd_df = cd_df.drop_duplicates()
cd_df.describe()
```

The timestamp duplicates issue persists. Let's check if they are actual duplicates or just various songs played at the same time over various feeds:


```python
# To check for timestamp related duplicates
timedupes = cd_df.duplicated(subset=['Stamp', 'Track', 'Artist', 'Feed_Id']).sum()
timedupes
```

Now that we know that we have no more possible duplicate entries we can move on to the feed station mapping data.


```python
fsm_df.info()
fsm_df.head(5)
fsm_df.describe()
```

The 'Feed_Id' is one of the main parameters for the analysis. It's worth the time to look more into the feed IDs over both datasets before the analysis starts.

From the feed station mapping dataset we see that there are 1125 feeds for 1122 stations. No duplicate rows are found in the feed station mapping data. This means that each 'Feed_Id' is tied to at least a radio station. Also up to 6 stations (depending on the combinations: 1x 4-to-1 or 3x 2-to-1 or 1x 3-to-1 & 1x 2-to-1) have a many-to-one feed mapping.

Also important to note is that the number of unique 'Feed_Id' in the consumption data is 1479 i.e. 354 more than the feed_ids supplied in the feed station mapping data. This means that these feed IDs won't have a station mapped to them (or to be precise, the radio station name is not known).

Going back to the consumption data, we see that there are more unique 'Feed_Name' entries than 'Feed_Id'. Since all columns in cd_df have non-null values, this means that some feed IDs are connected to multiple feed names.

How many unique feed IDs are connected to more than 1 feed name?


```python
# Aggregate to find how each Feed_Id is connected to each Feed_Name.
feedidsnames = cd_df.groupby(['Feed_Id']).agg({'Feed_Name': 'nunique'}).reset_index()

# Feed_ids with more than 1 feed_name
multinamefeedids = feedidsnames[feedidsnames['Feed_Name']>1]
len(multinamefeedids)
```

There are 141 unique feed IDs connected to more than 1 feed name. Since there is a discrepancy in the number of feed IDs provided in the feed station mapping and the feed IDs in the consumption data, let's look into how many of these multiname feed IDs are in the consumption data:


```python
# To check how many of these unique Feed Ids are in the consumption data
check = cd_df[cd_df['Feed_Id'].isin(multinamefeedids['Feed_Id'].values)].drop_duplicates(['Feed_Id','Feed_Name']).sort_values(by=['Feed_Id'])
len(check)
```

A total of 322 feed_ids in the consumption data have more than one name mapped to them.

How many of them are in the feed station mapping data?


```python
# To check if the multi name Feed_Ids are in the feed station mapping dataset
test = fsm_df[fsm_df['Feed_Id'].isin(multinamefeedids['Feed_Id'].values)]
len(test)
```

This quick check shows that none of the multi name feed_ids are in the feed station mapping data. 

The datasets will be merged on the 'Feed_Id' column. This will allow for an easy distinction of the station name. When the station name is missing, it is suggested to default to the feed info. This can mean either the 'Feed_Id' or 'Feed_Name' values. 

Reverting to the 'Feed_Name' has the issue of increasing the number of duplicates. This happens because some feed IDs without a parent station are connected to more than one feed name. Think of it as different listeners tuning in to the same feed, the listeners being the feed names. 

Since we look at the plays at feed level, and 'Feed_Id' is what is used to connect to the stations, it is a better choice to default to the 'Feed_Id' value.


```python
# Merge
df = cd_df.merge(fsm_df, on = 'Feed_Id', how = 'outer').drop_duplicates()

# Add a name for the unnamed stations
df.loc[df['SMP_Station'].isnull(),'SMP_Station'] = df.loc[df['SMP_Station'].isnull(),'Feed_Id']

# Convert to datetime format
df['Stamp'] = pd.to_datetime(df['Stamp'], format='%Y-%m-%d %H:%M:%S %z UTC')
```

There is a request to "not double count plays coming from feeds that belong to the same station". The many-to-one relationship between feeds and stations means that songs can be played at a specific timestamp, on a specific radio station but on a different feed. 

In other words, the need is to distinguish a song airplay between multiple listeners. Royalty distribution is based on how many times songs are played from a source rather than how many people listened to them.


```python
# How many duplicates
dupes = len(df) - len(df.drop_duplicates(subset=['Stamp', 'Track', 'Artist', 'SMP_Station']))
dupes_pct = 100*((len(df) - len(df.drop_duplicates(subset=['Stamp', 'Track', 'Artist', 'SMP_Station'])))/len(df))

# Data without the duplicates
df = df.drop_duplicates(subset=['Stamp', 'Track', 'Artist', 'SMP_Station'])
dupes
```

We can double check this by counting the sum of the unique values when the dataset is grouped by 'SMP_Station' i.e. the number of unique stations. Indeed the number of rows is the same as with df after the duplicate drop, so everything checks out.


```python
# Create check df
df_check = df
df_check['Combined columns'] = df['Stamp'].astype(str) + df['Track'] + df['Artist'] + df['Feed_Id']
print(df_check.groupby(['SMP_Station'])['Combined columns'].nunique().sum() == len(df))

# To drop the test column
df = df.drop(['Combined columns'], axis=1)
```

To make the dataframe a bit more intuitive, a 'Song' column is added. The date is broken down in 'Year', 'Month', 'Day' and 'Time' columns that will be interesting to use as filters for the song play data visualizations. Once this is done, we can take a look at our dataset with *.describe()*:


```python
# Song column
df['Song'] = df['Artist'] + ' - ' + df['Track']

# Datetime columns
df['Date'] = df['Stamp'].dt.date
df['Year'] = df['Stamp'].dt.year
df['Month'] = df['Stamp'].dt.month
df['Day'] = df['Stamp'].dt.day
df['Time'] = df['Stamp'].dt.time

# Dataframe overview
df.info()
df.head(5)
df.describe(datetime_is_numeric=True, include='all')
```

The 'Song' column has 8 unique values. This means that we have one song per artist. This is typical for radio shows for copyright issues. However, there have been some exceptions to this, like that time John Peel played back to back Teenage Kicks by The Undertones at Radio 1 in 1978 (Terry Hooley, who released the single, wouldn't have any copyright claims most probably).

It's interesting to have a snapshot of the song/artist distribution per station feed, as well as the number of feeds per station. Using *.groupby(), .nunique() and .value_counts()* shows that 98.4% of the radio stations only have 1 feed.


```python
# Airplay % per artist per feed per station
airtistperstation = df.groupby(['SMP_Station', 'Feed_Id'])['Artist'].value_counts(normalize=True)*100

# Number of feeds per station in %
feedsperstation = df.groupby('SMP_Station')['Feed_Id'].nunique().value_counts(normalize=True)*100
```

## Visualizations & discussion

Now we can complement the analysis with interactive visualizations and provide some insights and commentary about the song radio play data. We will focus on:

- Airplay per artist
- Airplay per station
- Airplay over time (year, month, day of the month, hour of the day)

Since the data span a bit less than 2 years, it is easier to use the year as a first breakdown dimension. Besides, yearly statistics are very common in the music industry.

First we create some dataframes with grouped data in order to break down and categorize the separate visualizations.


```python
### Visualization data

# Unique artists played per station per year
aps = pd.DataFrame(df.groupby('SMP_Station')['Artist'].nunique()).reset_index()
aps = aps.rename(columns={'Artist':'artists'})
apsy = pd.DataFrame(df.groupby(['SMP_Station', 'Year'])['Artist'].nunique()).reset_index()
apsy = apsy.rename(columns={'Artist':'Artists'})
# Total artists played per station
taps = pd.DataFrame(aps['artists'].value_counts()).reset_index().rename(columns={'index':'tot_artists','artists':'stations'})
tapsy = pd.DataFrame(apsy['Artists'].value_counts()).reset_index().rename(columns={'index':'tot_artists','artists':'stations'})
# Artist airplay
aa = pd.DataFrame(df.groupby('Artist')['Track'].value_counts())
aa = aa.rename(columns={'Track':'counts'}).reset_index()
aa['Song'] = aa['Artist'] + ' - ' + aa['Track']
# Songs per year
spy = pd.DataFrame(df.groupby('Song')['Year'].value_counts())
spy = spy.rename(columns={'Year':'Plays'}).reset_index()
# Songs per month and year
spmy = pd.DataFrame(df.groupby(['Song', 'Year'])['Month'].value_counts())
spmy = spmy.rename(columns={'Month':'Plays'}).reset_index()
# Songs per day and year
spd = pd.DataFrame(df.groupby('Song')['Day'].value_counts())
spd = spd.rename(columns={'Day':'Plays'}).reset_index()
spdy = pd.DataFrame(df.groupby(['Song', 'Year'])['Day'].value_counts())
spdy = spdy.rename(columns={'Day':'Plays'}).reset_index()
# Hourly plots and resampling per year
time_df = pd.DataFrame(df.groupby(['Year', 'Time'])['Song'].value_counts())
time_df = time_df.rename(columns={'Song':'Plays'}).reset_index()
time_df['Time'] = pd.to_datetime(time_df['Time'], format='%H:%M:%S') # Includes wrong date but is not important
resampled_tot = pd.DataFrame(time_df.resample('60min', on='Time', label='right')['Plays'].sum()).reset_index()
resampled_tot['Time'] = resampled_tot['Time'].astype(str).str[11:]
resampled_2019 = pd.DataFrame(time_df.query('Year==2019').resample('60min', on='Time', label='right')['Plays'].sum()).reset_index()
resampled_2019['Time'] = resampled_2019['Time'].astype(str).str[11:]
resampled_2020 = pd.DataFrame(time_df.query('Year==2020').resample('60min', on='Time', label='right')['Plays'].sum()).reset_index()
resampled_2020['Time'] = resampled_2020['Time'].astype(str).str[11:]
# Songs over a year by date
date_df = pd.DataFrame(df.groupby(['Year', 'Date'])['Song'].value_counts())
date_df = date_df.rename(columns={'Song':'Plays'}).reset_index()
date_df['Date'] = pd.to_datetime(date_df['Date'])
```

Now we have what we need to start plotting. Let's begin with how many airtime each song had each year. By clicking the legend, you can select each song. Hovering over the lines gives the exact number of plays for that specific day. These plots give us an overview of how the airplays were distributed for each of the two years.


```python
# For the legend order
songs = df['Song'].unique().tolist()
legend = songs.sort()

# Plot
fig = px.line(date_df.query('Year==2019'), 
              x='Date', 
              y='Plays', 
              color='Song',
              category_orders={'Song': songs})
fig.update_layout(title_text='Plays per day in 2019')
fig.show()

import plotly.io as pio

pio.write_html(fig, file='figure1.html', auto_open=True)

fig = px.line(date_df.query('Year==2020'),
              x='Date',
              y='Plays', 
              color='Song',
              category_orders={'Song': songs})
fig.update_layout(title_text='Plays per day in 2020 (until September)')
fig.show()
```

<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/1.png" alt="linearly separable data">

Some outliers immediately show up. In 2019, the top artists ('artist_03' and 'artist_07') have a massive drop in airplay in the period 02/15-02/18 & 02/25. Let's look into the station activity during that period.



```python
activefeeds19 = df[(df.Stamp >= '2019-02-15') & (df.Stamp <= '2019-02-18')]['SMP_Station'].count()
activestations19 = df[(df.Stamp >= '2019-02-15') & (df.Stamp <= '2019-02-18')]['SMP_Station'].nunique()
activefeeds20 = df[(df.Stamp >= '2020-02-15') & (df.Stamp <= '2020-02-18')]['SMP_Station'].count()
activestations20 = df[(df.Stamp >= '2020-02-15') & (df.Stamp <= '2020-02-18')]['SMP_Station'].nunique()
print('2019 Feb 15 - Feb 18 | Feeds:',activefeeds19, '| Stations:', activestations19)
print('2020 Feb 15 - Feb 18 | Feeds:',activefeeds20, '| Stations:', activestations20)
```

During that period the reported station activity was significantly reduced compared to the next year. Asimilar reduction in airtime is observed for the other artists as well. Due to the lower overall plays they have, it is not as impactful as with artists 03 and 07.

For 2019 there is no significant decrease in airtime over the course of the year. For 2020 however, the data stops at the end of September. Given that the song airtime reporting has been consistent through the previous year, we expect similar levels of plays for 2020 too. This is also the reason for the reduced total plays for 2020, as 3 months of data can make a lot of difference.

What's more interesting is the share of each song/artist in the yearly plays. This is easier to see in a donut chart. 


```python
### Airplays, per year and total

plays2019 = spy.query('Year==2019')['Plays'].sum()
plays2020 = spy.query('Year==2020')['Plays'].sum()
playstotal = plays2019 + plays2020

fig = make_subplots(rows=1, cols=3, specs=[[{'type':'domain'}, {'type':'domain'}, {'type':'domain'}]])

fig.add_trace(go.Pie(labels=spy['Song'].drop_duplicates(), 
                     values=spy.loc[(spy['Year'] == 2019), ['Plays']]['Plays'], 
                     name='2019',
                     sort=False,
                     hole=.4),
              1, 1);

fig.add_trace(go.Pie(labels=spy['Song'].drop_duplicates(), 
                     values=spy.loc[(spy['Year'] == 2020), ['Plays']]['Plays'], 
                     name='2020',
                     sort=False,
                     hole=.4),
              1, 2);

fig.add_trace(go.Pie(labels=spy['Song'].drop_duplicates(), 
                     values=aa['counts'], 
                     name='Total',
                     sort=False,
                     hole=.4),
              1, 3);

fig.update_layout(legend_title_text='Artist & song',
                  legend=dict(
                  yanchor='top',
                  y=0.75,
                  xanchor='left',
                  x=1.1),
                  title_text='Airplays 2019, 2020 and total',
                  # Add annotations in the center of the donut pies.
                  annotations=[dict(text='2019', x=0.11, y=1, font_size=18, showarrow=False),
                               dict(text=str(plays2019), x=0.095, y=0.5, font_size=18, showarrow=False),
                               dict(text='2020', x=0.5, y=1, font_size=18, showarrow=False),
                               dict(text=str(plays2020), x=0.5, y=0.5, font_size=18, showarrow=False),
                               dict(text='Total', x=0.89, y=1, font_size=18, showarrow=False), 
                               dict(text=str(playstotal), x=0.905, y=0.5, font_size=18, showarrow=False)])
fig.show()
```

The number of plays for each donut chart is displayed in its hole. It looks like two songs dominated the airtime for both 2019 & 2020. It is also interesting that the airplay percentages for all songs, has remained the same across the two years. Steady number of plays over such a long time suggests that the songs are title (or credits or intermission) tracks for radio shows or podcasts. 

Let's now check the song variety per station i.e. how many stations played more than 1 song. 


```python
### Unique artists played per station

stations2019 = apsy.query('Year==2019')['SMP_Station'].nunique()
stations2020 = apsy.query('Year==2020')['SMP_Station'].nunique()
stationstotal = apsy['SMP_Station'].nunique()

fig = make_subplots(rows=1, cols=3, specs=[[{'type':'domain'}, {'type':'domain'}, {'type':'domain'}]])

fig.add_trace(go.Pie(labels=tapsy['tot_artists'], 
                     values=apsy.loc[(apsy['Year'] == 2019), ['Artists']]['Artists'].value_counts(), 
                     name='2019', 
                     hole=.4),
              1, 1);

fig.add_trace(go.Pie(labels=tapsy['tot_artists'], 
                     values=apsy.loc[(apsy['Year'] == 2020), ['Artists']]['Artists'].value_counts(), 
                     name='2020', 
                     hole=.4),
              1, 2);

fig.add_trace(go.Pie(labels=tapsy['tot_artists'], 
                     values=tapsy['Artists'], 
                     name='Total', 
                     hole=.4),
              1, 3);

fig.update_layout(legend_title_text='Number of artists',
                  legend=dict(
                  yanchor='top',
                  y=0.75,
                  xanchor='left',
                  x=1.1),
                  title_text='Unique artists played per station',
                  # Add annotations in the center of the donut pies.
                  annotations=[dict(text='2019', x=0.11, y=1, font_size=18, showarrow=False),
                               dict(text=str(stations2019), x=0.11, y=0.5, font_size=18, showarrow=False),
                               dict(text='2020', x=0.5, y=1, font_size=18, showarrow=False),
                               dict(text=str(stations2020), x=0.5, y=0.5, font_size=18, showarrow=False),
                               dict(text='Total', x=0.89, y=1, font_size=18, showarrow=False), 
                               dict(text=str(stationstotal), x=0.885, y=0.5, font_size=18, showarrow=False)])
fig.show()
```

In the center hole of each donut, is the number of active stations. Also note that for this set of charts, the color key corresponds to artist count rather than song name as in the rest of the graphs.

First thing noticeable is an increase in the station count. In 2020, 73 new stations started playing the 8 song catalogue. 

If you're looking for variety, you'll be disappointed as ~45% of the stations chose to play only one song. This however shows that this specific song gets most airtime and it further adds to the guess of it being the theme of a very popular podcast/radio show.

Let's look more into how the plays are distributed over calendar months and days of the month. Again, the visualizations will be divided by year.


```python
# Total airplay per day of the month
fig = px.bar(spdy.query('Year==2019'), 
             x='Day',
             y='Plays', 
             color='Song', 
             title='Airplay per day of the month - 2019')

fig.update_layout(xaxis = dict(tickmode = 'linear', 
                 tick0 = 1, dtick = 1)
                 )
fig.show()

fig = px.bar(spdy.query('Year==2020'),
             x='Day',
             y='Plays', 
             color='Song',
             title='Airplay per day of the month - 2020')

fig.update_layout(xaxis = dict(tickmode = 'linear', 
                 tick0 = 1, dtick = 1)
                 )
fig.show()
```

Plotting the total airplays (no distinction between songs now) over a 24 hour day period would show the listening (or airtime reporting) behaviour. This graph is closer to what one would expect from a listener: decreased listeners/feed activity during afternoon/night compared to steady activity during working hours. 


```python
### Plays per hour per year and total

fig = go.Figure()

fig.add_trace(go.Scatter( 
              x=resampled_tot['Time'],
              y=resampled_tot['Plays'],
              mode='lines',
              name='Total'))
fig.add_trace(go.Scatter( 
              x=resampled_2019['Time'],
              y=resampled_2019['Plays'],
              mode='lines',
              name='2019'))
fig.add_trace(go.Scatter( 
              x=resampled_2020['Time'],
              y=resampled_2020['Plays'],
              mode='lines',
              name='2020'))

fig.update_layout(title='Number of radio plays per hour of the day',
                   xaxis_title='Hour',
                   yaxis_title='Plays')

fig.show()
```

As a final note, it would be interesting to have some data on how long each song was played for each feed and radio station. Having actual airplay duration data can paint a better picture for the consumption vs artist revenue challenge.
