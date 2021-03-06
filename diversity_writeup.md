Data
----

This is a bit of exploratory analysis of a dataset measuring the diversity within U.S. counties. Source: Mike Johnson Jr. who got the data from census.gov and posted it on [Kaggle](https://www.kaggle.com/mikejohnsonjr/us-counties-diversity-index). The diversity index runs on a scale from 0-1. **Long story short**: the closer a county is to 1, the more diverse it is. **Long story medium**: the diveristy index of a county represents the odds that two randomly selected people in the county will be of the same race. **Long story long**: find the score by summing the square of the proportion of each race relative to the population of the county as a whole. Then subtract this number from 1.

Dataset cleaning / polishing
----------------------------

To do this brief analysis we'll enter the [Hadleyverse](https://www.r-bloggers.com/welcome-to-the-hadleyverse/), and load the `tidyr`, `dplyr`, `stringr` and `ggplot2` packages.

``` r
library(tidyr)
library(dplyr)
library(stringr)
library(ggplot2)

diversityindex <- read.csv("https://raw.githubusercontent.com/NeonTheorist/Diversity.Index/master/diversityindex.csv")
```

The data are clean but could be polished some more. The main issue is that the counties and states are not in separate columns. Let's fix that.

``` r
#get rid of non-counties from dataset
counties <- diversityindex[grep(",", diversityindex$Location),]

#separate Location column into County and State columns
counties <- separate(data = counties, col = Location, into = c("County", "State"), sep = ",")

#get rid of white spaces in State names
counties$State <- str_trim(counties$State)

#get rid of DC
counties <- filter(counties, State != "DC")

#rename awkwardly named columns
new_names <- c("County", "State", "Diversity.Index", "Black", "AmInd.Alaska", "Asian", "NatHaw.PacIsl", "TwoOrMore", "Hisp.Lat.", "White")
names(counties) <- new_names
dim(counties)
```

    ## [1] 3142   10

``` r
head(counties)
```

    ##                       County State Diversity.Index Black AmInd.Alaska
    ## 1 Aleutians West Census Area    AK        0.769346   7.4         13.8
    ## 2              Queens County    NY        0.742224  20.9          1.3
    ## 3                Maui County    HI        0.740757   0.8          0.6
    ## 4             Alameda County    CA        0.740399  12.4          1.2
    ## 5     Aleutians East Borough    AK        0.738867   7.7         21.8
    ## 6              Hawaii County    HI        0.738772   0.8          0.6
    ##   Asian NatHaw.PacIsl TwoOrMore Hisp.Lat. White
    ## 1  31.1           2.3       4.8      14.6  29.2
    ## 2  25.2           0.2       2.7      28.0  26.7
    ## 3  28.8          10.6      23.3      10.7  31.5
    ## 4  28.2           1.0       5.2      22.7  33.2
    ## 5  41.4           0.7       3.7      13.5  12.9
    ## 6  22.1          12.7      29.5      12.2  30.7

Good enough. We have ten columns with seven representing the proportion of the population representing a particular racial group. With 3142 observations, we have a pretty decent set to work with. Thanks, America, for being so big and countyful.

Basic analysis
--------------

First things first. Let's start out by ranking and sorting states in terms of their overall diversity and plotting them on a graph from least to most diverse. If states have roughly the same level of diversity, we should see a horizontal line. Otherwise the line should slope upward.

``` r
state <- counties %>%
  group_by(State) %>% 
  summarise(State.Diversity = mean(Diversity.Index)) %>%
  arrange(State.Diversity) %>%
  mutate(Rank = rank(State.Diversity))

ggplot(state, aes(x = Rank, y = State.Diversity)) + geom_point()
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-3-1.png?raw=true)

Clearly one state is wacky.

``` r
tail(state, 1)
```

    ## # A tibble: 1 x 3
    ##   State State.Diversity  Rank
    ##   <chr>           <dbl> <dbl>
    ## 1    HI       0.7184228    50

Darn you, Hawaii. While some of the most interesting data are outliers, they're also annoying when doing preliminary analysis. So for the sake of simplicity and increased clarity, let's pretend that Hawaii's [independence movement](https://en.wikipedia.org/wiki/Hawaiian_sovereignty_movement) succeeds and drop it from out dataset.

``` r
state_49 <- filter(state, !State == "HI")
ggplot(state_49, aes(x = Rank, y = State.Diversity)) + geom_point() + geom_smooth()
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-5-1.png?raw=true)

``` r
ggplot(state_49, aes(x = State.Diversity)) + geom_histogram(breaks = seq(0.1, 0.6, by = 0.05), col = "black", fill = "blue", alpha = 0.3) + geom_density(col = 2) + scale_y_continuous(breaks = seq(0,11, by = 1 )) +
  labs(title = "State Level Diversity Index Distribution", x = "Diversity Index", y = "Number of States")
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-5-2.png?raw=true)

``` r
summary(state_49$State.Diversity)
```

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##  0.1009  0.1816  0.2719  0.3019  0.4379  0.5158

So the median diversity of the 49 states is 0.27 and the mean is 0.3, indicating that there's a right-tailed skew. This is not surprising when we look at this wacky bimodal distribution.

Diversity of counties within states
===================================

The main question I had when I came across the dataset was whether there was an interesting trend *between* states with respect to the diversity between counties *within* states.

Huh?

Here's what I mean. I currently live in California, a pretty diverse state on average. But is each county in California about as diverse as each of the other California counties? In other words: is there a lot of deviation in terms of variance between counties within a given state?

Let's find out with a couple of plots! Actually, the second one is just a magnification of the obnoxiously grouped portion of the main plot.

``` r
counties %>%
  filter(!State == "HI") %>%
  group_by(State) %>%
  summarise(avg = mean(Diversity.Index), spread = sd(Diversity.Index)) %>%
  ggplot(aes(x = avg, y = spread, label = State)) + geom_text(size = 3, alpha = 0.9) + geom_smooth() + geom_smooth(method = "lm", col = "rosybrown1", se = F, alpha = 0.1) +
  labs(title = "State Diversity: Mean and Variations Between Counties", x = "State Diversity Mean", y = "Standard Deviation Between Counties") 
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-6-1.png?raw=true)

``` r
counties %>%
  filter(!State == "HI") %>%
  group_by(State) %>%
  summarise(avg = mean(Diversity.Index), spread = sd(Diversity.Index)) %>%
  ggplot(aes(x = avg, y = spread, label = State)) + geom_text(size = 3, alpha = 0.9) + geom_smooth() +
  labs(title = "State Diversity: Mean and Variations Between Counties", x = "State Diversity Mean", y = "Standard Deviation Between Counties") +
    coord_cartesian(xlim = c(0.12,0.22))
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-6-2.png?raw=true)

These plots measure the standard deviation of county diversity within each state against that state's mean county diversity. Vermont, for instance, has a low state diversity and low diversity between counties. New Jersey has a very high mean and high diversity between counties. South Carolina has a high mean diversity rating but a much lower deviation between counties than the trendline would expect.

I want to make sure that I'm clear on what the standard deviation represents here (and I'll elaborate more later below): a high standard deviation means that as you move from one county to another, you encounter very different odds that randomly selecting two people from within that county will result in two people being of the same race.

How should we interpret the trend? There is quite a bit of deviation, but the general shape is a parabola, which is represented by the blue line. This shouldn't be too surprising. Logically, a state with a mean of zero would have only homogeneous counties, meaning that things don't get more or less diverse as you switch counties, making the standard deviation zero. The same goes for a state comprised of perfectly diverse counties. Since things don't get less diverse as you move from county to county, the standard deviation falls to zero.

The straight line is the trendline if we select a linear model to represent the trend. Using this line would obviously be a big mistake in this case. Moral of the story: linear models are great, but it's a good idea to look at the data before announcing that higher state diversity indexes are associated with higher rates of deviation between counties.

Looking at each state individually, it's clear that there are very different trends within individual states.

``` r
counties %>%
  filter(!State == "HI") %>%
  group_by(State) %>%
  mutate(median_diversity = median(Diversity.Index)) %>%
  arrange(median_diversity) %>%
  ggplot(aes(x = State, y = Diversity.Index)) + geom_boxplot() +
  theme(axis.text.x = element_text(size = 6))
```

![](https://github.com/NeonTheorist/Diversity.Index/blob/master/unnamed-chunk-7-1.png?raw=true)

The boxplot reveals differences that the scatterplot hides. Looking at the zoomed-in scatterplot, we can see that Nevada and Wisconsin are very similar in terms of mean diversity index and standard deviation, but the states look different when we examine the boxplot. Wisconsin's median diversity score is lower than Nevada's but Nevada has a greater number of outlier counties. Wisconson's counties are mostly clustered around the median, except for one county with a greater diversity score than any county in either of the two states.

Caveat
------

As it stands, there's an interesting implication here. Remember that the measure is looking at diversity over the county level, and diversity amounts to "if I took two people at random from the county, how likely is it that they would be of the same race?"

The weird thing is that a state with a homogeneous population would have the **same** diversity index score (and standard deviation between counties) than a state that was perfectly segregated at the county level. That's because if we had a state that had 5 counties that were 100% White and counties that were 100% African-American, any two people we selected from any individual county would always have the same race. For example, Vermont, Maine and New Hampshire all have very low diversity scores and standard deviations between counties, but this does not imply that these states are populated almost exclusively by people of one race. It could be that citizens there segregate to an extreme degree into distinct counties.

### What next?

Given the caveat, the next bit of analysis that interests me would be to look at the proportion of racial groups within each county of each state, while being especially mindful of comparing states that appear close together on the scatterplot above. More specifically:

-   Do states with low diversity scores have mostly homogeneous populations, or is it a case of segregation?
-   Do states with similar scores have the same proportion of racial groups?
-   To what extent to states with similar overall racial demographics differ in terms diversity score?

Fun questions - definitely fun enough to keep me busy with a dataset that I initially thought was pretty dull.
