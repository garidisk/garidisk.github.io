---
title: "Radio station feed data analysis - Part II"
date: 2021-08-22
tags: [data visualization, python, plotly, interactive plots]
header:
  image: "/images/radioclash.jpg"
excerpt: "Data Visualization, Python, Plotly"
mathjax: "true"
classes: wide
---

_This is Part II of the radio station feed data analysis. This part contains only the visualizations and data interpretation segments of the analysis._

_The first part can be found [here](https://garidisk.github.io/radiofeed-pt1-eda/)._

_The full notebook can be found [here](https://drive.google.com/file/d/12hSI_T7zoPcxEId7Mrb-KCTXNY_syMFT/view?usp=sharing). This version also supports interactive plots which cannot be rendered here (due to .html file size constraints)._

_The header image is the cover of the 2013 release of the ["The Clash Hits Back"](https://www.discogs.com/The-Clash-Hits-Back/master/594585) compilation. The same photo has also been used in a 1981 promo release called ["If Music Could Talk"](https://www.discogs.com/The-Clash-Hits-Back/master/594585), which includes interviews with the band members._

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
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/2.png" alt="linearly separable data">


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
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/3.png" alt="linearly separable data">

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
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/4.png" alt="linearly separable data">

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
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/5.png" alt="linearly separable data">
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/6.png" alt="linearly separable data">

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
<img src="{{ site.url }}{{ site.baseurl }}/images/radiofeed/7.png" alt="linearly separable data">

As a final note, it would be interesting to have some data on how long each song was played for each feed and radio station. Having actual airplay duration data can paint a better picture for the consumption vs artist revenue challenge.
