NOAA Data Analysis: Effects on Population Health and Economy
=====================================================================
This document analyzes storm data taken from NOAA. The data used in the analysis can be obtained [here](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2) and documentation on the data [here](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf). The goal of this analysis is to answer two questions:

1. Across the United States, which types of events (as indicated in the EVTYPE variable) are most harmful with respect to population health?
2. Across the United States, which types of events have the greatest economic consequences?

#Data Processing
First, the dataset is read in from the .csv. This script will be using the dplyr package to manipulate the dataset. 

```r
library(dplyr)
dat <- read.csv("repdata-data-StormData.csv.bz2")
df <- tbl_df(dat)
```
The dataset needs to be preprocessed. This will convert the column EVTYPE to lowercase and trim its white space, filter out rows after 2003, and combine redundant EVTYPE labels. The rows are more relevant after 2003 because reporting was not standardized and as accurate before.

```r
df$EVTYPE <- tolower(df$EVTYPE)
df$EVTYPE <- gsub("^\\s+|\\s+$", "", df$EVTYPE)
df$BGN_DATE <- as.Date(df$BGN_DATE, "%m/%d/%Y %H:%M:%S")
df <- df %>%
    filter(BGN_DATE > as.Date("2003", "%Y")) %>%
    filter(EVTYPE != "astronomical high tide") %>%
    filter(EVTYPE != "landslide") %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="heavy surf/high surf", 
                          "high surf")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="hurricane/typhoon", 
                          "hurricane")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="marine thunderstorm wind",
                          "marine tstm wind")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="sleet storm", "sleet")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="storm surge/tide", 
                          "storm surge")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="thunderstorm wind", 
                          "tstm wind")) %>%
    mutate(EVTYPE=replace(EVTYPE, EVTYPE=="winter weather/mix", 
                          "winter weather"))
df$EVTYPE <- factor(df$EVTYPE)
```
The first question to answer is which type of event is most harmful to population health. The variables that address population health are FATALATIES and INJURIES. 

```r
library(dplyr)
byEvent <- group_by(df, EVTYPE)
pop_health <- summarize(byEvent, 
                        FATALITIES=mean(FATALITIES, na.rm=TRUE),
                        INJURIES=mean(INJURIES, na.rm=TRUE))
```
Top 5 fatalities on average:

```r
head(arrange(pop_health, desc(FATALITIES)), 5)
```

```
## Source: local data frame [5 x 3]
## 
##           EVTYPE FATALITIES  INJURIES
## 1        tsunami  1.6500000 6.4500000
## 2    rip current  0.7900000 0.4600000
## 3 excessive heat  0.6781293 3.0907840
## 4      hurricane  0.5508475 8.2203390
## 5      avalanche  0.5062241 0.3609959
```
Top 5 injuries on average:

```r
head(arrange(pop_health, desc(INJURIES)), 5)
```

```
## Source: local data frame [5 x 3]
## 
##           EVTYPE FATALITIES INJURIES
## 1      hurricane  0.5508475 8.220339
## 2        tsunami  1.6500000 6.450000
## 3 excessive heat  0.6781293 3.090784
## 4           heat  0.3229901 1.723554
## 5        tornado  0.0788936 0.913327
```
Since there can be different number of observations for each event type, looking at the average instead of the total seems to be a better metric.

Here is a graph of number of deaths and injuries per event:

```r
library(ggplot2)
```

```
## Find out what's changed in ggplot2 with
## news(Version == "1.0.1", package = "ggplot2")
```

```r
ph_graph <- summarize(byEvent, total=mean(FATALITIES+INJURIES))
ph_graph <- arrange(ph_graph, total)
ph_graph$ordered <- reorder(ph_graph$EVTYPE, desc(ph_graph$total))
ggplot(ph_graph, aes(x=EVTYPE, y=total)) + 
    geom_bar(stat = "identity", aes(x=ordered)) +
    theme(axis.text.x = element_text(angle = 60, hjust = 1)) +
    ggtitle("Effects on Population Health by Event") + 
    xlab("Event Type") + 
    ylab("Number of Deaths/Injuries per Event")
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png) 
The top 3 event types that total the most deaths/injuries on average are hurricane, tsunami, and excessive heat.

The next question to answer is what impact does each event have on economy. The variables that address this are PROPDMG and CROPDMG. First, the data needs to be multiplied by its corresponding factor PROPDMGEXP and CROPDMGEXP.

```r
byEvent$PROPDMGEXP <- toupper(byEvent$PROPDMGEXP)
byEvent$CROPDMGEXP <- toupper(byEvent$CROPDMGEXP)
byEvent <- 
    filter(byEvent, PROPDMGEXP %in% c("K","M","B")) %>%
    filter(CROPDMGEXP %in% c("K","M","B")) %>%
    mutate(PROPDMG = ifelse(PROPDMGEXP == "K", PROPDMG * 10^3,
                            ifelse(PROPDMGEXP == "M", PROPDMG * 10^6,
                            ifelse(PROPDMGEXP == "B", PROPDMG * 10^9,
                            PROPDMG)))) %>%
    mutate(CROPDMG = ifelse(CROPDMGEXP == "K", CROPDMG * 10^3,
                            ifelse(CROPDMGEXP == "M", CROPDMG * 10^6,
                            ifelse(CROPDMGEXP == "B", CROPDMG * 10^9,
                            CROPDMG))))
```
Now that the damages have their respective factors, reporting can be done. Again, the mean is a better metric than the sum since there can be many more of one event type than the other.

```r
damages <- summarize(byEvent, PROPDMG=mean(PROPDMG), CROPDMG=mean(CROPDMG))
```
Top 5 property damages types on average:

```r
damages <- arrange(damages, desc(PROPDMG))
head(damages, 5)
```

```
## Source: local data frame [5 x 3]
## 
##           EVTYPE   PROPDMG      CROPDMG
## 1      hurricane 442543864 42248224.242
## 2    storm surge  32689845     5985.915
## 3          flood  10032096   261649.739
## 4        tsunami   7582211     1052.632
## 5 tropical storm   2480866   964253.807
```
Top 5 crop damages types on average:

```r
damages <- arrange(damages, desc(CROPDMG))
head(damages, 5)
```

```
## Source: local data frame [5 x 3]
## 
##           EVTYPE      PROPDMG    CROPDMG
## 1      hurricane 4.425439e+08 42248224.2
## 2 tropical storm 2.480866e+06   964253.8
## 3 excessive heat 2.662619e+03   934345.4
## 4   frost/freeze 9.275930e+03   911742.7
## 5        drought 2.480411e+04   516574.5
```
Here is a graph of total economic damages per event:

```r
econ_graph <- summarize(byEvent, total=mean(PROPDMG + CROPDMG) / 10^6)
econ_graph <- arrange(econ_graph, total)
econ_graph$ordered <- reorder(econ_graph$EVTYPE, desc(econ_graph$total))
ggplot(econ_graph, aes(x=EVTYPE, y=total)) + 
    geom_bar(stat = "identity", aes(x=ordered)) +
    theme(axis.text.x = element_text(angle = 60, hjust = 1)) +
    ggtitle("Effects on Economy per Event") + 
    xlab("Event Type") + 
    ylab("Economic Damages per Event (Million USD)")
```

![plot of chunk unnamed-chunk-11](figure/unnamed-chunk-11-1.png) 
The top 3 event types that total the most economic damages are hurricane, storm surge, and flood.
Comparing Economic Damage vs. Population Health:

```r
final <- merge(ph_graph, econ_graph, by = "EVTYPE")
ggplot(final, aes(x=total.x, y=total.y)) + 
    geom_point() + 
    geom_text(aes(label=ifelse(total.x>1, as.character(EVTYPE),
                               ifelse(total.y>10, as.character(EVTYPE),
                                      ""))), 
              angle=45, size=3) +
    ggtitle("Economic Damage vs. Population Health") +
    xlab("Average Number of Deaths/Injuries") +
    ylab("Average Economic Damage (Million USD)")
```

![plot of chunk unnamed-chunk-12](figure/unnamed-chunk-12-1.png) 

#Results
From this analysis, the most dangerous event type is a hurricane averaging the most economic damage and population deaths/injuries. Other dangerous events that average more than 1 death/injury or more than $10 million in economic damages are:

- hurricane
- typhoon
- excessive heat
- heat
- rip current
- storm surge
- flood
