---
title: "Radio station feed data analysis - Part I"
date: 2021-08-22
tags: [data wrangling, data science, messy data]
header:
  image: "/images/radioclash.jpg"
excerpt: "Data Wrangling, Data Analysis, Data Visualization, Python"
mathjax: "true"
---
_This is Part I of the radio station feed data analysis. This part contains only the data wrangling part of the exploratory data analysis._

_Visualizations and result analysis are included in the second part, which can be found [here](https://garidisk.github.io/radiofeed-pt2-viz/)._

_The full notebook can be found [here](https://drive.google.com/file/d/12hSI_T7zoPcxEId7Mrb-KCTXNY_syMFT/view?usp=sharing)._

_Header image is the cover of the 2013 release of the ["The Clash Hits Back compilation"](https://www.discogs.com/The-Clash-Hits-Back/master/594585). The same photo has also been used in a 1981 promo release called ["If Music Could Talk"](https://www.discogs.com/The-Clash-Hits-Back/master/594585), which included band member interviews._

## Context
Everytime a song is played on the radio, its airplay is reported to a performing rights organization. There are many such organizations, depending on the country/continent the radio station is based on. One common issue with the logging of the airtime is in the data entry itself. Duplicate entries, erroneous song naming, missing names in general etc. Therefore, data cleaning is even more important when dealing with radio play data.

## Introduction
This project deals with song play data from various radio feeds. They span a period of 21 months from January 2019 to September 2020. Two datasets are been used for this project: a set containing song radio play information and another containing mapping information between feeds and stations. Using the supplied datasets, songs can be pinned to their respective feeds and feeds to their respective radio stations. The focus of this notebook will be on the following:

- Cleaning the dataset
- Visualization and insights 

The data cleaning will focus on identifying and removing duplicate entries. Duplicate entries will have an impact in the revenue stream towards the artists.


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

