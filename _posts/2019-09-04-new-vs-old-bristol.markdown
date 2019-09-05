---
layout: post
title:  "New vs Old Bristol"
date:   2019-09-04 20:32:05 -0400
categories: Sports
---

Bristol is a high-banked half-mile bullring that has provided some of the most exciting moments in NASCAR history. However, in recent years fans haven’t been as happy with the racing.  In 2007, Bristol Motor Speedway moved to progressive banking, which basically means the banking is steeper as you get closer to the wall.
This has opened up more racing grooves making it easier for drivers to pass, where as prior when the only groove was the bottom the drivers would result to “bumping and banging” to move other cars from their path to make a pass. Fans for years have loved the action this single groove created, but now with the multiple grooves and the drivers being able to pass without having to move another driver out of the way the accidents and excitement seems to be gone in the eyes of a lot of fans. But let’s examine the data and see if the numbers match the eye test.

This post will use data gathered from [racing reference](https://www.racing-reference.info/) to determine, statistically, if the racing has changed much since the addition of progressive banking or not. Unfortunately there is no API or data export available from [racing reference](https://www.racing-reference.info/) so in order to gather the data we need to scrape the site first. Begin by first importing the necessary libraries.


```python
import pandas as pd
import numpy as np
import requests
import re
from decimal import Decimal
from bs4 import BeautifulSoup
```

We can use requests and beautiful soup in order to get the pages we need to crawl and to parse them for the data we need. Since we’re going to be fetching multiple pages it makes sense to create a function for requesting the pages and it will simply return a soup page object.


```python
def get_page(url):
    domain = 'https://racing-reference.info'
    full_url = F"{domain}{url}"

    r = requests.get(full_url)
    soup = BeautifulSoup(r.text, 'lxml')

    return soup
```

The next step is requesting the track page for Bristol and then all the rows for the races we'll be looking at. The page is set up as nested tables, so using the browser's inpsector tool you can see the table containing the Cup Series races is the 7th one down. The first night race at Bristol was in 1978 and is the second race at Bristol each season, so we can calculate the starting index for the rows as the 36th and we want every other row.


```python
# get the bristol race listing page
page = get_page('/tracks/Bristol_Motor_Speedway')

# the 7th table down is the cup series race table
table = page.find_all('table')[7]

# all the race rows from 1978
# (the first night race) going
# forward to the 2019 race.
rows = table.find_all('tr')[36::2]
```

Now that the race rows have all been obtained it's time for the fun stuff, actually looking at the data. This post will examine caution breakdown and lead changes for all the races prior to and after the progressive banking being added in 2007 (referred to from here on out as pre_pb and post_pb.) As well, the analysis will occur on the overall data for all the races in both times and on a per race basis for both times.

A method for getting caution breakdown table for each race will be required here, so let's write that now.


```python
# this method returns the
# caution breakdown table
# for the race as a dataframe
def caution_breakdown(page):
    """
    Method to take a beautiful soup race page and return
    the caution breakdown table as a dataframe. Not all
    race pages contain the caution table, if it doesn't
    just return an empty dataframe.
    """
    try:
        # the table isn't in the same index on every page, using
        # match argument in pandas read_html method we can actually
        # find the table that contains the 'Caution flag breakdown:'
        # header regardless of it's position
        table = pd.read_html(str(page), match='Caution flag breakdown:')[1]

        # we don't need the flag icon column
        # or the reason and free pass columns
        # since we're only looking at green flags
        drop_columns = [0, 4, 5]
        table.drop(table.columns[drop_columns], axis=1, inplace=True)

        # add friendly column names
        table.columns = ['from_lap', 'to_lap', 'number']

        # slice the dataframe from the index by 2 leaving
        # off the last row.
        return table.iloc[2:-1:2].astype({
            'from_lap': 'int64',
            'to_lap': 'int64',
            'number': 'int64'
        })
    except Exception as e:
        return pd.DataFrame()
```

Next we're going to create 4 dataframes (pre_pb per-race/overall and post_pd per-race/overall.) The reason we do this is because when looking at averages, which is what we'll be using for analysis here, you don't want averages of averages instead you want the average of the whole and then we can drill down into each race and see how the overall breaks down.


```python
        # these are going to be our
        pre_pb_per = []
        pre_pb_overall = []
        post_pb_per = []
        post_pb_overall = []
        for row in rows:
            # relative path to the race page
            race_url = row.find('td').find('a')['href']

            # regular expression to extract year from url.
            year = re.search(r'/race/(\d{4})_', race_url).group(1)

            # lead changes are the last TD element in the row.
            # Make sure this is converted to an int.
            lead_changes = int(row.find_all('td')[-1].text)

            # the number of caution flags is alredy on the main
            # table as well, 4th from the end.
            num_cautions = int(row.find_all('td')[-4].text)

            # individual race breakdown
            data = {
                'year': year,
                'lead_changes': lead_changes,
                'num_cautions': num_cautions
            }

            # get the race page as a dataframe
            df = caution_breakdown(get_page(race_url))

            # if the caution breakdown dataframe is
            # empty, then go ahead and set run values
            # we'd put in the race data to nan.
            if df.empty:
                avg_run = np.nan
                last_run = np.nan
            else:
                avg_run = df['number'].mean()
                last_run = df.iloc[-1]['number']

            data.update({
                'avg_run': avg_run,
                'last_run': last_run
            })

            if int(year) >= 2007:
                post_pb_per.append(data)
                post_pb_overall.append(df)
            else:
                pre_pb_per.append(data)
                pre_pb_overall.append(df)

        # contact our overall lists of dataframe's to create
        # the overall dataframes for each time period.
        pre_pb_overall_df = pd.concat(pre_pb_overall)
        post_pb_overall_df = pd.concat(post_pb_overall)

        # to create the per race data frames just pass the list
        # of race data dictionaries to the dataframe constructor.
        pre_pb_per_df = pd.DataFrame(pre_pb_per)
        post_pb_per_df = pd.DataFrame(post_pb_per)

        # make sure that the missing caution flag
        # data is filled in with the mean of the
        # respective columns
        pre_pb_per_df['avg_run'].fillna(pre_pb_per_df['avg_run'].mean(), inplace=True)
        pre_pb_per_df['last_run'].fillna(pre_pb_per_df['last_run'].mean(), inplace=True)
```


## Overall Data:

### Pre Progressive Banking:
1. **Average green flag run: 33.53979238754325**
2. **Average final run length: 73.78260869565217**
3. **Average number of lead changes: 14.0**
4. **Average number of cautions: 10.827586206896552**

### Post Progressive Banking:
1. **Average green flag run: 43.5**
2. **Average final run length: 55.46153846153846**
3. **Average number of lead changes: 16.615384615384617**
4. **Average number of cautions: 8.923076923076923**

### How they vary:
1. **Percentage change in average run length: 23.544208906289462**
2. **Percentage change in average final run length: -33.03382982572514**
3. **Percentage change in average number of lead changes: 15.740740740740748**
4. **Percentage change in average number of cautions: -21.3436385255648**

## Conclusion
The number of overall cautions is down since the addition of the progressive banking, but the final run length on average has been shorter, meaning shorter runs to the finish and more exciting finishes. Another thing to point out is that lead changes are up and so is overall run length. While this investigation doesn't lend itself to finding statistical answers to why that is, one could surmise that longer runs, means green flag stops, which could cycle leaders. An interesting thing to investigate.
Overall after the eye test and the trend analysis I would conclude that while "New Bristol" isn't the same as it once was, it's still a very exciting track putting on great racing with lots of action.



