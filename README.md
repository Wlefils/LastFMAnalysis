# Project Overview
I love listening to music. I’ve listened to a lot of music over the years, and much of this listening history has been tracked by apps like Spotify. I always get excited for Spotify's Year Wrapped feature, but I didn’t want to wait until the end of the year to get my results. Spotify Wrapped is also inherently limited in that it doesn’t allow you to compare different time periods, rather than simply the last year (unless you have your previous year's results saved). Thankfully, I’ve also been using Last.fm for several years, and racked up tens of thousands of “scrobbles” (another word for “plays” or “listens”) on that platform. I decided to play around with the **Last.FM API** and turn my personal music listening history into an analytics project.

**The end goal of this project was to create an animated bar chart displaying how my top 20 most listened to artists shifted and changed over time.** Specifically, I was looking at data from May 2016 (when I created my Last.fm account) until December 2022.

### Tools used:
* **RStudio** (ggplot2, gganimate, tidyverse, tidyr, dplyr, lubridate, and more)
* **Excel**

### Data Acquisition

The first step was to use an R package called [scrobbler](https://cran.r-project.org/web/packages/scrobbler/index.html) to retrieve my personal data. To do this, I had to go to Last.FM's developer site and request an API key. Using the **download_scrobbles()** function with this package, I downloaded my listening history as a data frame with the following code (replace "YourUsername" etc with your actual details if you'd like to replicate this project).

```R
# Pulling my listening data from the API
username <- "YourUsername"
apiKey <- "YourAPIkey"
apiSecret <- "YourAPIsecret"
lastFM <- download_scrobbles(username = username, api_key = apiKey)
```
Because I had over six years’ worth of listening data (over 72,000 scrobbles), it took a while to download. I made sure to save my data frame to avoid having to repeat this step if I cleared my environment later (which I did, so I was glad to have this backup point). This is the code I used for this step:
```R
# Saving the file to avoid redownloading later
write.csv(lastFM, file = "lastFMfromAPI.csv")
```

### Data Cleaning
Last.FM's API provided a lot of additional data for each scrobble that wasn’t necessary for the scope of this project. To make the dataset a bit easier to work with, I subset the dataframe to retain only the specific variables I planned on using: song title, artist name, album title, and the date I listened to each track.

```R
# subset to only the relevant data - song, artist, album, date
lastFM2 <- lastFM[,c(2,4,6,9)]
```

The next step was to examine the current state of the data using the head(function), as shown below.

![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/5de8d6dd-228b-4d3a-aa44-9ef44490d5a8)


At this time, I was focused on setting up the fields to suit my needs. The date column initially had unnecessary values (for this analysis) of time and day, whereas I was only interested in grouping my data by month. The best way I know of to do this is to create a variable in the dataframe that contains the month. First, I wanted to clean up the date column by removing the timestamp. I did this with the lubridate package. I also used lubridate to change the date column to the date data type.

```R
# remove time from the output
lastFM2$date <- gsub(",.*","",lastFM2$date)    # regex - removes everything after a comma (inclusive of comma)
```

```R
# Converting the data type to date
lastFM2$date <- lubridate::dmy(lastFM2$date) #dmy as date is ordered Day/Month/year
```

Next, I sorted the data by the newly improved date column.

```R
# Order from oldest to newest
lastFM2 <- lastFM2[order(lastFM2$date),]
```

 To add the running total, I used the **group_by()** and **mutate()** functions from the dplyr package (this is why it was important to ensure the data was in the right order).

```R
#Let's add the running total number of scrobbles for each artist
library(dplyr)
lastFM2 <- lastFM2 %>% group_by(artist) %>%
  mutate(count=row_number())
```

After using the **head()** function once again, I verified to see that I'd successfully added the Count column to the dataframe. This column served as the running total for tracking the number of plays each artist had at any given point in time. To demonstrate this more clearly, I used the **head()** function to call a specific artist, which returned a list of their songs in order of when they were listened to, with the the running total increasing by one for each additional scrobble. 

![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/fd54e2cc-effc-4000-aeaa-6db849e1e2c8)



At this point, I split the date into year and month, to facilitate data processing.

```R
# Time to split the date into separate year and month columns
lastFM2$year <- year(lastFM2$date)
lastFM2$month <- month(lastFM2$date)
```

The main reason to split the date up like this is to easily create a new column called monthID. This is what allowed me to differentiate between the same month in different years (e.g., May 2016 vs May 2017). Without doing this, all months of the same name would be the same. Adding the monthID column resolved this by incrementing over time. This is also a critical component in the animated bar chart at the end.

```R
# Add monthID column so same month in different years has unique identifier
lastFM2$monthID <- lastFM2$month + ((lastFM2$year - 2016)*12)
```
```R
# Change date to remove day variable
lastFM2$date <- format(lastFM2$date, format="%m-%y")

# Grouped by month
lastFMGrouped <- group_by(lastFM2, artist, monthID, date) %>%
  summarise(count = max(count))

# Order the grouped dataframe in chronologicall order
lastFMGrouped <- lastFMGrouped[order(lastFMGrouped$monthID),]
```

![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/4e43451a-93d2-4c54-9c20-31aa4ab81e7c)

This is when I ran into a major issue. The above image shows a glimpse of the total scrobbles for Carly Rae Jepsen during a several month time period. I didn't listen to CRJ in January or February of 2017 (monthID 13 and 14, respectively). If I created the animated bar chart with the data as it was here, CRJ would be present in December of 2016, but disappear completely for the next two months, before reappearing in March of 2017. That's obviously not what I was looking for. My solution to this involved a multi-step process:

First, I made a list of each artist, date, and monthID in the existing dataset and combined them to create a new dataframe. The goal here was to make an entry for each artist for every month. Since my data contained 5,330 artists across 83 months, the new dataframe was comprised of 458,990 rows (5,330 * 83).

```R
#  Find every artist name and all monthIDs and dates
allArtists <- unique(lastFMGrouped$artist)
allMonthIDs <- unique(lastFMGrouped$monthID)
allDates <- unique(lastFMGrouped$date)

# Find the length for each of these and determine row numbers in a new dataframe.
> length(allArtists)
> length(allMonthIDs)
> length(allDates)

# Create a new dataframe with all artists, inputting 0 plays for each month
newDF <- data.frame(artist = rep(allArtists, 83),          # add every artist (length(allArtists)) times 
                    monthID = rep(allMonthIDs, 5530),       # add every monthID (length(allArtists)) times
                    date = rep(allDates, 5530),             # add every date (length(allArtists)) times
                    count = rep(0,458990))                  # set count = 0 for all of these
```

With this completed, the next step was to combine this new dataframe and lastFMGrouped. I used dplyr's **full_join()** function to do so. Then I grouped the data again (as was done previously) and exported it to a csv for further cleaning in Excel.

```R
# Combining this with grouped dataframe
allArtistsAllDates <- full_join(lastFMGrouped, newDF)

# Grouping by artist and month/date again
allArtistsAllDates <- group_by(allArtistsAllDates, artist, monthID, date, .drop = F) %>%
  summarise(count = max(count))
  
# Exporting to csv
write.csv(allArtistsAllDates, file = "allArtistsAllDates.csv")
```

## Data Cleaning in Excel

### Initial Formatting
With the data in Excel, it was time to do some further cleaning and processing. I created a column called "maxcount" to calculate the highest number of plays each artist had up to that date. The formula I used for that calculation is below
```Excel
= IF(B3=B2, IF(E3=E2, F2,MAX(E2:E3)),0)
```
![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/52f6873b-bde2-4b91-b5fa-08c409fbed64)

I'll explain what each part of this formula does, step by step:
```
=IF(B3=B2,   # If the artist name does not change
IF (E3 = E2, # Identify if the count column increased from the previous value
F2,          # If it has not changed, retain the previous maximum value
MAX(E2:E3)   # If it has changed, adjust it to the higher value
),0)         # If the artist name changes, reset to 0
```

Note that in the screenshot above the date column is shown as 16-May, 16-Jun, and so on. The date was formatted improperly as dd/mm. This is because the actual values entered are 05/16 and 06/16, so Excel assumed this particular date format. To correct this, I had to use a simple =DATE formula. In cell G2, I used the following formula (and double clicked the fill handle to fill every row with it):

```
=DATE(2016, C2, 1) 
```

This formula formats the date in YMD format, and prints a date value that is the first day of each month of 2016, based on the monthID. I then copy and pasted this column (as values) in column D (date) to format the dates as needed. I also deleted the F column after I was done to ensure there were no empty or null values when I imported it back into R.


### Removing Excess Rows

At this stage, my file had over 500,000 rows, and I wanted to make it a bit smaller. Many of the rows were actually unnecessary - artists didn't need a row with a count of 0 in months before they received their first play. For example, I didn't listen to Twice until the 33rd monthID; I didn't need to retain the nearly 30 rows of Twice with a count of zero, I only wanted the rows where the count column was at least 1. My method of doing this was to filter the data to show rows with a maxcount of 0 and then delete them. Deleting hundreds of thousands of rows made my Excel freeze for a few minutes, but it did eventually complete this task - you may have different results with a more powerful PC. This reduced the number of rows by about two-thirds. Then, I turned off my filter and began the most tedious (but critically important) portion of the data cleaning process.


### Additional Cleaning Steps
My data required a significant amount of additional cleaning and processing before taking it back into R and creating the visualization. I ran into a few major issues. The first was that I listen to a lot of non-English language music, particularly Korean and Japanese artists. Trying to retrieve artist names, track titles, and album titles with a mix of kanji, katakana, hiragana, and Hangul proved quite challenging. When imported into Excel, they weren’t properly displayed, which often made it difficult to determine which artist these scrobbles belonged to. In most cases, these artists had some song titles that I could identify (due to English characters). In these instances, I was able to use Find and Replace to change their artist name to a single, standardized, English version. There were a few situations where I was forced to trawl through my listening history on the Last.FM website to get more information (like album art) or use Google Translate to identify the artist properly. I created a pivot table in Excel for all artists whose names suffered from this issue, and manually corrected as many as possible.

![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/09f59c74-ebed-49b5-aab2-b123f629ac33)

A second issue stemmed from my modality of listening to music. While Spotify has a native feature for linking your listening history to your Last.FM account, I also listen to a lot music on YouTube. I had downloaded an extension several months prior to this project that automatically scrobbled YouTube videos. The extension was useful but had issues of its own. The main problem with the extension is that it resulted in a lot of false positives – it would frequently mistake non-music content for songs. It also scrobbled every “chapter” in a video as its own track, which added a lot of extra plays for supposed songs that were just lengthy podcasts, tutorials, or other content. While these problems made the overall dataset less clean, they didn’t directly impact the limited scope of this project, as none of the false positives had an affect on my top 20 most listened artists. However, a similar issue that did have material impact on the project was that the extension would often mix up artist and song titles (reversing them or concatenating them inappropriately). While Spotify will sometimes have multiple variants of one track (in the cases of featured artists or remixes), on YouTube, there are often numerous different versions of a track: music videos, lyric videos, audio-only versions, live performances, and, in the case of K-pop and J-pop, fan translations and line distribution videos. All of these would be scrobbled separately, and often times the artist would be misidentified (either swapping the track for the artist or using a different string in the video’s title as the artist name). There wasn’t an easy way around this due to the myriad of different errors. It was difficult to determine the exact scope of the problem, but I knew it must have affected thousands of scrobbles. I opted to identify any artists in my top 50 most played according to Last.fm (thus basing it off the unclean version of the data from the API) and look for errors related to any of their top 20 songs. 

<img width="510" alt="Find_and_Replace_GG" src="https://github.com/Wlefils/LastFMAnalysis/assets/98787088/2a1ab4be-da96-43c5-b281-4164ed3f0107">

For example, Girls’ Generation (aka SNSD) is one of my top 50 most listened to artists (but ultimately not in my top 20). Due to this group being commonly referred to by several different name variants, I knew there would be a large number of unattributed scrobbles that rightfully belonged to them. I found the character string that corresponded to Girls’ Generation and used Excel’s Find and Replace function to properly attribute and standardize dozens of plays to them. 

In the same vein, K-pop music videos are frequently titled according to the following naming scheme “SONG NAME M/V”, with “M/V” being an abbreviation for “music video”. I had over 300 scrobbles with “M/V” in the title, and these were often attributed to an incorrect artist  (such as “Official” or something along those lines), but I manually corrected these problems.

<img width="617" alt="Find_and_Replace_MV" src="https://github.com/Wlefils/LastFMAnalysis/assets/98787088/9aae34bd-7aa7-45f4-a3c4-c8a63138b7ba">

Ultimately, I was satisfied with the data cleaning I was able to achieve via these methods. While not every row of data was completely cleaned, I was confident that I’d rectified the errors material to this project. It simply wouldn’t have been practical to manually clean every row of data, and wouldn’t have materially impacted the final result. 

After completing the cleaning, it was time to bring the data back into R and move forward with the visualization process.

## Finalizing in R

At this stage, I read the cleaned csv file back into R and created a table to display the top 20 most played artists for each month. A key point to note here is that this table uses the previously calculated running total, so it is displaying the Top 20 Most Played Artists up to that point, not the Top 20 Most Played Artists during a specific month. 

```R
# loading in cleaned and edited CSV from Excel
lastFMGrouped2 <- read.csv(file = "cleanedGrouped.csv", stringsAsFactors = F)

# Creating a table for the plot
# Retrieves the Top 20 Most Played Artists each month and adds a ranking column
lastFmTable <- lastFMGrouped2 %>%
  group_by(monthID) %>%
  mutate(rank = min_rank(-maxcount) *1) %>%       # multiply by 1 to change type to numeric (from integer) - This makes it so gganimates places it in between integer places when animated.
  filter(rank <= 10) %>%
  ungroup()
```

I also added a rank column to determine each artist's place in the top 20 for any given month; this served as the x-variable for the plot. Then, I reordered the dataframe by MonthID once more.
```R
#Reordering by month one more time
lastFmTable <- lastFmTable[order(lastFmTable$monthID),]
```

There was some additional adjustment necessary. I added another column called ordering, which was the inverse of the rank variable (i.e., where rank = 20, order = 1 and where rank = 19 ordering = 2). I also have the full code for the project here for more information on how everything was done. Many of the artist names in my Top 20 were too long and often overlapped each other, especially when the bars were moving around so quickly. To alleviate this, I used the **str_wrap** function from the **stringr** package. Another way to work around this would be to manually add line breaks for specific artists using the **gsub** function.

```R
# Adding the Ordering column - inverse of rank column
# Use ggplot(aes(rank, ...)) for biggest value at left
scrobblesTable <- lastFmTable %>%
  group_by(monthID) %>%
  mutate(ordering = min_rank(maxcount) * 1.0) %>%
  ungroup()
```

Below is the final code for the ggplot and the moving bar chart.
```R
plot <- scrobblesTable %>%
    ggplot(aes(ordering,
               group = artist)) +
    geom_tile(aes(y = maxcount/2,
                  height = maxcount,
                  width = 0.5,
                  fill = artist),
              alpha = 0.9,
              colour = "Black",
              size = 0.75) +
    # text on top of the bars
    geom_text(aes(y = maxcount, label = str_wrap(artist, width = 10)), size = 8,
              vjust = -0.5) +
    # label displaying the Date in the top left corner
    geom_label(aes(x=1.5,
                   y=4000,
                   label=date),
               size=12,
               color = 'black') +
    coord_cartesian(xlim = c(min(scrobblesTable$ordering) -1,max(scrobblesTable$ordering) +1),
    ylim = c(0, 1.1 * max(scrobblesTable$maxcount)),
    clip = "off",
    expand = TRUE) +
    labs(title = 'My most played artists over time.',
         subtitle = "My top LastFM artists from May 2016 through December 2022.",
         caption = '\n \n\nSource: LastFM | plot by Will LeFils',
         x = '',
         y = '') +
    theme(
        plot.title = element_text(size = 26),
        plot.subtitle = element_text(size = 20),
        plot.caption = element_text(size = 20),
        axis.ticks.x = element_blank(),
        axis.text.x  = element_blank(),
        axis.text.y = element_text(size = 12, angle = 90)
        legend.position = "none",
    ) +
    transition_states(monthID,
                      transition_length = 20,
                      state_length = 2,
                      wrap = F) +             # wrap = F prevents the last frame being an in-between of the last and first months
    ease_aes('cubic-in-out')
    animate(plot, 
            duration = 60, 
            fps = 20, 
            detail = 20, 
            width = 1920, 
            height = 1080, 
            end_pause = 90, 
            renderer = gifski_renderer())
anim_save("lastFMTop20.gif")
```

This code produced the animation seen below:

https://github.com/Wlefils/LastFMAnalysis/assets/98787088/451a32b5-018c-4e74-b55e-cb6cb08e922c

## Conclusion

### Insights
The biggest takeaway for me from this analysis is how significantly my music taste shifted starting in April 2020. **In March 2020, there were zero non-English artists in my top twenty**. Starting around this time, I began listening to so much K-pop that by August 2020 Twice and Blackpink had become my third and fourth most listened artists of all time, with (G)I-DLE reaching seventh, overcoming over four years of plays for artists like Lana Del Rey and Grimes who had previously been near the top spots. By December of 2020 Blackpink and Twice had supplanted Charli XCX and Pale Waves as my most played artists. Despite the release of a new Pale Waves album causing them to briefly overtake Twice, Twice would quickly leapfrog Blackpink to become my most-played artist beginning in May 2021. At this point, the gap just continued to grow between Twice, the distant second of Blackpink, and all of the other artists. **Non-English artists (specifically Korean and Japanese) comprised three of my top five and eight of my top twenty most played artists by December 2022**.


I also thought it was worth comparing what Last.FM's API (i.e., the uncleaned version of the data) lists as my Top 20 Most Played artists when compared to the final output of the bar chart (ensuring that the time periods in question are the same). At first glance, the two are quite similar. The top four artists are all in the same order relative to each other. However, on the bar chart, (G)-IDle is at rank five, while Last.FM places them in sixth. Similarly, "League of Legends" (representing all music produced by Riot Games save K/DA, who are represented with a separate bar) is at rank 8, whereas on Last.FM's rankings this artist is placed at fifteenth. Kinoko Teikoku, a Japanese shoegazing band, ranks thirteenth on the bar chart, but only at 27th according to Last.FM. Taylor Swift ranks sixteenth on Last.FM, but isn't included on my final bar chart (ranking just outside the top 20).


![image](https://github.com/Wlefils/LastFMAnalysis/assets/98787088/2302f739-6f07-4bf1-b6ca-d66b9cd98ab4)




### External Links
I want to thank Tom MacNamara for his excellent blog post [here](https://www.r-bloggers.com/2020/04/learning-gganimate-with-the-lastfm-api/), detailing a very similar project that inspired me. I also looked at some posts on Stackoverflow that provided some more details on how to improve the animation of the final chart, which can be found [here](https://stackoverflow.com/questions/52623722/how-does-gganimate-order-an-ordered-bar-time-series/52652394#52652394) and [here](https://stackoverflow.com/questions/53162821/animated-sorted-bar-chart-with-bars-overtaking-each-other/53163549#53163549).

* [Portfolio Homepage](https://wlefils.github.io/)
* [Resume](https://wlefils.github.io/Will%20LeFils%20Resume.pdf)
* [LinkedIn Profile](https://www.linkedin.com/in/will-lefils-57b838132/)
