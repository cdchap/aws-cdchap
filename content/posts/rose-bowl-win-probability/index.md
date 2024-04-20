---
title: 'Rose Bowl Win Probability'
date: 2024-04-03T21:41:02-05:00
draft: false
tags:
- football
categories:
- data analysis
---

**TLDR:**
Create a plot that shows the win probability throughout the 2024 Rose Bowl Game using data from [College Football Database API](https://collegefootballdata.com/). You can [click here]({{< relref "#full-code" >}}) to go to the full code.

## Introduction
What follows is the step by step approach that I took to plot the win probability for both teams in the 2024 Rose Bowl Game. The task is pretty straight forward. The api provides the exact data needed for the plot and one could make this plot with one api call. However I did seek to add a little more to the plot that I believe enhances the representation. Things like using the team colors for the lines, adding the final score with logos, and using quarter endings as the x tick marks. To achieve this there was additional data needed and a little extra work that needed to be done.

The end result of this work is shown below:

{{< figure src="rose-bowl-2024.png" title="2024 Rose Bowl Win Probability Chart" >}}

The data used for this analysis is from the [College Football Database API](https://collegefootballdata.com/). I am sure that the process could be done more efficiently. I am learning process, and sometimes I wanted just to try things. For example at one point I rename a column from "play_id" to "id". Not necessary, but I wanted to try out the `.rename()` method in Pandas. There are other such examples in the code.
## Libraries
There are just three libraries needed to create the plot. The `import` statements are show below. Note the additional statements regarding `matplotlib`. These imports are needed for the logos and final score annotation that is shown. The plot can be understood without that information, it is something a little extra.

Where `pandas` and `matplotlib` are both common libraries, the `cfbd` library may not be. It provides a convenient way to work with the api, but first it needs to be installed.

```bash
pip install cfbd
```

Then the imports.

```python
import pandas as pd
import cfbd
import matplotlib.pyplot as plt
from matplotlib.offsetbox import OffsetImage, AnnotationBbox
import matplotlib.image as mpimg
```
## Configure CFBD 
In order to use the `cfbd` library it needs to be configured with an api key. The keys are free to obtain and can be requested from [here](https://collegefootballdata.com/key). Once you have your key you can use the below code to configure.

```python
configuration = cfbd.Configuration()
configuration.api_key['Authorization'] = 'your-api-key'
configuration.api_key_prefix['Authorization'] = 'Bearer'

api_config = cfbd.ApiClient(configuration)
```

Now that it is configured we can use it to make a call to the api to get some data.
## Load Teams
One of the features that I wanted to include in the plot was to have the lines representing the different teams be the correct team colors. This is one of the values that is returned when a request is made to the `teams` api end point. To get all of the teams you can execute the below code.

*NOTE: there are other parameters if you want to narrow the api call*

```python
teams_api = cfbd.TeamsApi(api_config)
teams = teams_api.get_fbs_teams()
```

To see what the `teams` endpoint returns you can take a look at the [documentation](https://api.collegefootballdata.com/api/docs/?url=/api-docs.json#/). I do know that it returns a teams `color` and `alt_color`, so I made a teams data frame with those values, as well as the `id` and `school`.

```python
teams_df = pd.DataFrame.from_records([dict(id = t.id, school = t.school, color = t.color, alt_color = t.alt_color) for t in teams])
```

We will come back to the `teams_df` later. The next step is to find the data for the 2024 Rose Bowl
## Find Rose Bowl
For games we can use the `GamesApi` to get the data. This time we will add more parameters to the request, unlike the teams request where we just got all of the teams. The code is similar to the above teams request where the `GamesApi` is passed the configuration and then a call is made with the `get_games` method. This is where the parameters are passed. A year is required, so 2023 is passed. The default season type is 'regular', however we know that the Rose Bowl is a `postseason` game, so we set that as the `season_type`. With the parameters set this way we would get all of the post season games back from the api. Since we are only looking for the Rose Bowl we can narrow it down by passing a team we know that is in the game, so we pass the `team` parameter with a value of 'Michigan'. The api will now return all of the post season games(2) that Michigan was involved in. 

We will use this information to extract the Rose Bowl ID later to make a call to win probability and plays endpoints.

*__NOTE: if we passed Alabama as the team it would return just the Rose Bowl, making extracting the ID slightly easier, however I wanted to make a similar plot of the National Championship game for practice. Having both bowl games for Michigan would be helpful for that__*

As you will see below we create a data frame for the postseason games a little differently. Instead of specific data columns we just take in all of the data returned and create the data frame.

```python
games_api = cfbd.GamesApi(api_config)
postseason_um_games = games_api.get_games(year=2023, season_type='postseason', team='Michigan')
# DF of Michigan Post season games
postseason_um_games_df = pd.DataFrame.from_records([g.to_dict() for g in postseason_um_games])
```

The next step is to make a slice of the `teams_df` of the home and away teams of the Rose Bowl. We do this by creating a mask, and then filtering out those teams from the `teams_df`. The final bit of code in below section is to assign variables for the Rose Bowl ID `rose_bowl_id` and all the data from the `games_df` for the Rose Bowl-`rb`. These will be needed as parameters for the api calls for the win probability and play information respectively.

```python
rose_bowl_teams_mask = (teams_df['school'] == postseason_um_games_df['home_team'][0]) | (teams_df['school'] == postseason_um_games_df['away_team'][0]) 

rose_bowl_teams = teams_df[rose_bowl_teams_mask]

rose_bowl_id = postseason_um_games_df['id'][0]
rb = postseason_um_games_df.iloc[0]
```

## Rose Bowl Win Probability
Conveniently the api has an endpoint that returns the win probability, for the home team, for each play throughout the game. This is the data that is going to be shown on the plot. To get the data is a similar process as before. We pass our api configuration to the __MetricsApi__ this time, then use the `get_win_probability_data` method. We pass the `rose_bowl_id` ,that we assigned previously, as the `gam_id` parameter.

```python
#get win prob
probability_api = cfbd.MetricsApi(api_config)
rose_bowl_win_prob = probability_api.get_win_probability_data(game_id=int(rose_bowl_id))
```

With the data in hand we next create a data frame and I renamed the column `play_id` to just `id` because that is the column name that is used on the plays data frame that we will create soon. Eventually we will be merging these data frames together and it makes the code a little easier to understand in the merge statement if they both have the same name for these values.

```python
# Make a DF with the plays in the win prob
rb_wp_df = pd.DataFrame.from_records([p.to_dict() for p in rose_bowl_win_prob])
rb_wp_df = rb_wp_df.rename(columns={'play_id' : 'id'})
```

The MetricsApi is great, and returns all of the data we need, but there is one issue that needs to be dealt with. The plot I wanted to make had the win probability of both teams. Our data frame only has values for `home_win_prob`. To make a column called `away_win_prob` we use the following code:

```python
rb_wp_df['away_win_prob'] = 1 - rb_wp_df['home_win_prob'] 
```

That settles the win probability data frame for now. Soon we will merge with the play data frame that will be talked about next. The win probability data has the data that we want to plot, however it is missing when the plays happen within the context of the game clock. For this data we will go back to the api.

## Get Play by play
As mentioned earlier a crucial bit of data is not available from the Metrics api. That is, which quarter each play is in. To get this data we can make a call to an endpoint that gives us information about every play in the game. To achieve that we go through the same process as before, setting up the api and making a call. This the parameters are `season_type`, `year`, `week`, and `team`. For team we are using the variable we assigned the Rose Bowl data and selecting the home team, `rb['home_team']`, or "Michigan". Then we create a data frame with the returned data.

```python
# get data for play by play
play_api = cfbd.PlaysApi(api_config)
rb_plays = play_api.get_plays(season_type='postseason', year=2023, week=1, team=rb['home_team'])
rb_plays_df = pd.DataFrame.from_records([p.to_dict() for p in rb_plays])
```

The `rb_plays_df` has several columns we are not interested in. We could have omitted these columns with the same technique we created the `teams_df`. Fo this data frame I wanted to try passing a list of column names that we want.

```python
#filter df to have only the columns needed
filter_columns = ['id', 'play_number', 'period', 'clock']
rb_plays_df_filtered = rb_plays_df[filter_columns]
```

With the new data frame in place, there is another issue that needs to be dealt with. The `clock` column is still in a json format, and it contains the the quarter data that we need to make the x-axis ticks we need for our plot. To move this data into their own columns we use Pandas `json_noralize()` method and assign those to a variable. Then we we add those columns to the our data frame so that we have columns for `period`, `minutes`, and `seconds` (the key value pairs contained in the `clock` json). We use the Pandas `concat()` method for this, where we pass the data frame and clock variable in a list, and indicate that we want them to be columns with the axis parameter. Finally we drop the `clock` column, as it is no longer needed. 

```python
# normalize clock data and drop te clock column
clock = pd.json_normalize(rb_plays_df_filtered['clock'])
rb_plays_df_filtered = pd.concat([rb_plays_df_filtered, clock], axis=1)
rb_plays_df_filtered = rb_plays_df_filtered.drop('clock', axis=1)
```

Now our `rb_plays_df_filtered` data frame only contains the values we are interested in. The next step is to combine it with the `rb_wp_df` data frame so that we are dealing with one data frame for our plot. We can achieve this because both data frames have a play ID that are the same for each data frame.
## Merge the Data Frames
To merge the data frames we have been working with we use the Pandas `merge()` method and do a left join, keeping all of the plays from the win probability data frame. We do this because the win probability contains one more play than the plays data frame. That being the kick off, where the initial, or pregame, win probability value is.

```python
plays_df = pd.merge(rb_wp_df, rb_plays_df_filtered, how='left', on='id')
```

With the merge complete, we are nearly ready to plot our data. There is one issue that needs to be addressed however. The final play in our data frame has a NA value in the `period` column. To correct this we give it a value of 5. Why 5? This will come into play when we plot our data frame. Each play has a `period` value which is what quarter the the play took place in. The Rose Bowl went into overtime, so all plays in overtime have a value of 5. Since the final play took place in overtime, we give it the correct value.

```python
plays_df['period'] = plays_df['period'].fillna(5)
```

The final step is to filter to have only the columns we will need from the `plays_df`. This data frame (`plays_df_filtered`) is what we will use for our plot. I did leave some columns that are unused, in the event that I want to annotate the plot with play information, those columns being play_text, minutes, and seconds. 

```python
play_column_filter = ['id', 'play_text', 'home', 'away', 'home_score', 'away_score','home_win_prob', 'away_win_prob', 'period', 'minutes', 'seconds', 'play_number_x']

plays_df_filtered = plays_df[play_column_filter]
```
## Plot

### X Tick Marks
The plot I wanted to create was one that had quarter endings as tick marks, and have a label with something like "End of 1st". To achieve this I created a dictionary with numbers 0-5 as the key, and a string for the label. Next I filtered the data frame to have only the unique values for the period, which are 1-5 (this is the reason that we filled the NA with a value of 5 earlier) and created two empty lists.

Looping over the unique periods data frame slice we find the index of the last occurrence of each period. That is we find the last play of each quarter. That way the tick can be aligned with that play. When append the values of the index to the `xtick_position` and the value of the `period_labels` to the `xtick_labels`.

```python
# Create the dictionary to map period values to labels
period_labels = {0: 'start', 1: 'End 1st', 2: 'End 2nd', 3: 'End 3rd', 4: 'End 4th', 5: 'End OT'}

# Get unique period values
unique_periods = plays_df_filtered['period'].unique()
xtick_positions = []
xtick_labels = []

# Create tick positions and labels
for period in sorted(unique_periods):
    index = plays_df_filtered.loc[plays_df_filtered['period'] == period].index[-1]
    xtick_positions.append(index)
    xtick_labels.append(period_labels[period])
```

### The Plot
Now that we have the x-axis ticks and labels ready we can get to the business to plotting our data. Starting off with the FiveThirtyEight style and then moving on to assigning the size of the figure.

```python
plt.style.use('fivethirtyeight')

fig, ax = plt.subplots(figsize= (20, 12))
```

Next both the home and away team win probabilities are plotted, being sure to add the team color as the color of the line. In this case the alternate color was used for Michigan, as the blue and red (of Alabama) were too similar to be understood clearly.

```python
# Plot
ax.plot(plays_df_filtered.index, plays_df_filtered['home_win_prob'], color=rose_bowl_teams.iloc[1]['alt_color'], label='Michigan Win Probability')
ax.plot(plays_df_filtered.index, plays_df_filtered['away_win_prob'], color=rose_bowl_teams.iloc[0]['color'], label='Alabama Win Probability')
```

Next we set some labels so that consumers of the chart will know what they are looking at.

```python
ax.set_ylabel('Win Probability')
ax.set_title(f'{rose_bowl_teams.iloc[0]['school']} v. {rose_bowl_teams.iloc[1]['school']} 2024 Rose Bowl Win Probability')
```

I wanted to have the y-axis ticks to show percentage points at 20% intervals. Achieving this is less work than what was needed for the x-axis.

```python
yticks = [0, 0.2, 0.4, 0.6, 0.8, 1.0]
ytick_labels = [f"{int(100 * y)}%" for y in yticks]
ax.set_yticks(yticks)
ax.set_yticklabels(ytick_labels)
```

To finish of the the chart I added the x-axis ticks and labels made earlier and ensured that the legend was shown.

```python
# Set the x-axis ticks and labels
ax.set_xticks(xtick_positions)
ax.set_xticklabels(xtick_labels)

ax.legend()
```

The state of the plot at this point is reasonable enough to share. However I had the logos for the teams, provided by College Football Data's maintainer, and wanted to add a little more flare.
### The Annotation
Using the matplotlib `offsetbox` and `image` libraries I was able to achieve what I was looking to do. That is show the final score near the last play index with both the Michigan and Alabama logos.

Initially we load the images using their file path (I am using Jupyter Lab).

```python
# Load the logo images
logo1 = mpimg.imread('logos/Michigan.png')
logo2 = mpimg.imread('logos/Alabama.png')
```

Next is to create the OffsetImage objects

```python
# Create OffsetImage objects for the logos
image1 = OffsetImage(logo1, zoom=0.8)  # Adjust the zoom factor as needed
image2 = OffsetImage(logo2, zoom=0.8)
```

Next I gathered the data needed for the 2 annotation boxes, that is the final play index, win probabilities for both teams, and the scores for both teams.

```python
# Get the final play index and win probabilities
final_play_index = plays_df_filtered.index[-1]
home_win_prob = plays_df_filtered.loc[final_play_index, 'home_win_prob']
away_win_prob = plays_df_filtered.loc[final_play_index, 'away_win_prob']
final_home_score = plays_df_filtered.loc[final_play_index, 'home_score']
final_away_score = plays_df_filtered.loc[final_play_index, 'away_score']
```

To add the logos to the plot two annotation boxes were created with the data from above. Adjusting the `xybox` values to place them on the plot where I wanted them (small adjustments were made after the annotation for the final score was added).

```python
# Create AnnotationBbox objects for the logos and scores
ab1 = AnnotationBbox(image1, (final_play_index, home_win_prob), xybox=(-60, 52), frameon=False, xycoords='data', boxcoords='offset points', pad=0.5)
ab2 = AnnotationBbox(image2, (final_play_index, home_win_prob), xybox=(50, 52), frameon=False, xycoords='data', boxcoords='offset points', pad=0.5)
# Add the logos and scores to the plot
ax.add_artist(ab1)
ax.add_artist(ab2)
```

And the final piece to the plot, I added an annotation of the final score and nestled it between the logos.

```python
ax.annotate(f"Final\nMichigan: {final_home_score}\nAlabama: {final_away_score}", xy=(final_play_index, home_win_prob), xytext=(-5, 10), ha='center', textcoords='offset points')

plt.savefig('rose-bowl-2024.png')
```

## Conclusion
This was a fun exercise for me, and I would say that this is the first interesting plot that I have made worthy of sharing. I am sure that there are better techniques available to create a plot like this. I hope to learn them as I progress. 
## Full Code
```python
import pandas as pd
import cfbd
import matplotlib.pyplot as plt
from matplotlib.offsetbox import OffsetImage, AnnotationBbox
import matplotlib.image as mpimg

configuration = cfbd.Configuration()
configuration.api_key['Authorization'] = 'your-api-key'
configuration.api_key_prefix['Authorization'] = 'Bearer'

api_config = cfbd.ApiClient(configuration)

teams_api = cfbd.TeamsApi(api_config)
teams = teams_api.get_fbs_teams()

teams_df = pd.DataFrame.from_records([dict(id = t.id, school = t.school, color = t.color, alt_color = t.alt_color) for t in teams])

games_api = cfbd.GamesApi(api_config)
postseason_um_games = games_api.get_games(year=2023, season_type='postseason', team='Michigan')
# DF of Michigan Post season games
postseason_um_games_df = pd.DataFrame.from_records([g.to_dict() for g in postseason_um_games])

rose_bowl_teams_mask = (teams_df['school'] == postseason_um_games_df['home_team'][0]) | (teams_df['school'] == postseason_um_games_df['away_team'][0]) 
rose_bowl_teams = teams_df[rose_bowl_teams_mask]

rose_bowl_id = postseason_um_games_df['id'][0]
rb = postseason_um_games_df.iloc[0]
#get win prob
probability_api = cfbd.MetricsApi(api_config)
rose_bowl_win_prob = probability_api.get_win_probability_data(game_id=int(rose_bowl_id))

# Make a DF with the plays in the win prob
rb_wp_df = pd.DataFrame.from_records([p.to_dict() for p in rose_bowl_win_prob])
rb_wp_df = rb_wp_df.rename(columns={'play_id' : 'id'})

rb_wp_df['away_win_prob'] = 1 - rb_wp_df['home_win_prob'] 

# get data for play by play
play_api = cfbd.PlaysApi(api_config)
rb_plays = play_api.get_plays(season_type='postseason', year=2023, week=1, team=rb['home_team'])
rb_plays_df = pd.DataFrame.from_records([p.to_dict() for p in rb_plays])

# Filter to the columns I want to combine with win prob
filter_columns = ['id', 'play_number', 'period', 'clock']
rb_plays_df_filtered = rb_plays_df[filter_columns]
clock = pd.json_normalize(rb_plays_df_filtered['clock'])
rb_plays_df_filtered = pd.concat([rb_plays_df_filtered, clock], axis=1)
rb_plays_df_filtered = rb_plays_df_filtered.drop('clock', axis=1)

plays_df = pd.merge(rb_wp_df, rb_plays_df_filtered, how='left', on='id')
plays_df['period'] = plays_df['period'].fillna(5)

play_column_filter = ['id', 'play_text', 'home', 'away', 'home_score', 'away_score','home_win_prob', 'away_win_prob', 'period', 'minutes', 'seconds', 'play_number_x']

plays_df_filtered = plays_df[play_column_filter]

# Create the dictionary to map period values to labels
period_labels = {0: 'start', 1: 'End 1st', 2: 'End 2nd', 3: 'End 3rd', 4: 'End 4th', 5: 'End OT'}

# Get unique period values
unique_periods = plays_df_filtered['period'].unique()
xtick_positions = []
xtick_labels = []

# Create tick positions and labels
for period in sorted(unique_periods):
    index = plays_df_filtered.loc[plays_df_filtered['period'] == period].index[-1]
    xtick_positions.append(index)
    xtick_labels.append(period_labels[period])
    
plt.style.use('fivethirtyeight')
#plt.style.use('ggplot')

fig, ax = plt.subplots(figsize= (20, 12))

# Plot
ax.plot(plays_df_filtered.index, plays_df_filtered['home_win_prob'], color=rose_bowl_teams.iloc[1]['alt_color'], label='Michigan Win Probability')
ax.plot(plays_df_filtered.index, plays_df_filtered['away_win_prob'], color=rose_bowl_teams.iloc[0]['color'], label='Alabama Win Probability')
#ax.set_xlabel('Play Number')
ax.set_ylabel('Win Probability')
ax.set_title(f'{rose_bowl_teams.iloc[0]['school']} v. {rose_bowl_teams.iloc[1]['school']} 2024 Rose Bowl Win Probability')

yticks = [0, 0.2, 0.4, 0.6, 0.8, 1.0]
ytick_labels = [f"{int(100 * y)}%" for y in yticks]
ax.set_yticks(yticks)
ax.set_yticklabels(ytick_labels)


# Set the x-axis ticks and labels
ax.set_xticks(xtick_positions)
ax.set_xticklabels(xtick_labels)

ax.legend()

# Load the logo images
logo1 = mpimg.imread('logos/Michigan.png')
logo2 = mpimg.imread('logos/Alabama.png')

# Create OffsetImage objects for the logos
image1 = OffsetImage(logo1, zoom=0.8)  # Adjust the zoom factor as needed
image2 = OffsetImage(logo2, zoom=0.8)

# Get the final play index and win probabilities
final_play_index = plays_df_filtered.index[-1]
home_win_prob = plays_df_filtered.loc[final_play_index, 'home_win_prob']
away_win_prob = plays_df_filtered.loc[final_play_index, 'away_win_prob']
final_home_score = plays_df_filtered.loc[final_play_index, 'home_score']
final_away_score = plays_df_filtered.loc[final_play_index, 'away_score']

# Create AnnotationBbox objects for the logos and scores
ab1 = AnnotationBbox(image1, (final_play_index, home_win_prob), xybox=(-60, 52), frameon=False, xycoords='data', boxcoords='offset points', pad=0.5)
ab2 = AnnotationBbox(image2, (final_play_index, home_win_prob), xybox=(50, 52), frameon=False, xycoords='data', boxcoords='offset points', pad=0.5)
# Add the logos and scores to the plot
ax.add_artist(ab1)
ax.add_artist(ab2)

ax.annotate(f"Final\nMichigan: {final_home_score}\nAlabama: {final_away_score}", xy=(final_play_index, home_win_prob), xytext=(-5, 10), ha='center', textcoords='offset points')
plt.savefig('rose-bowl-2024.png')
```

### Technology
- [Jupyter Lab](https://jupyter.org/)
- [pandas](https://pandas.pydata.org/docs/)
- [matplotlib](https://matplotlib.org)
- [cfbd](https://github.com/CFBD/cfbd-python)