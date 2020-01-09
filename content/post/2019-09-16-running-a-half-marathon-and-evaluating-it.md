---
title: Running a half-marathon... and evaluating it
author: Florian Handke
date: '2019-09-22'
slug: running-a-half-marathon-and-evaluating-it
categories: []
tags:
- sport
- half-marathon
- reichenau
- '2019'
- running
---









Last week I ran my first half marathon on the island Reichenau near Konstanz. You don't win any prizes with 62nd place, but that wasn't my goal either. Since the organizer put the results of the race online, I took a closer look at the data.  I show the results in the following post.

The data I evaluate in this blog post are freely accessible at [SV Reichenau](https://www.svreichenau.de/images/leichtathletik/Strassenlauf/Ergenislisten/2019_Halbmarathon.pdf). 
Since I don't want to spread the names of the runners, I will extract them in this context, but not process them further.

As in the past, I like to use the `datapasta` package for fast data extraction. In this case, however, I was happy too early. The individual attributes are not separated and have to be separated first.

The record of me as an example:


```r
dplyr::glimpse(mens_result %>% 
          filter(str_detect(V1, "Handke")))
```

```
## Observations: 1
## Variables: 3
## $ V1 <chr> "62. 64 Handke"
## $ V2 <chr> "Florian 1991 Männer Haupt. 01:44:45"
## $ V3 <dbl> 3
```

The following packages were also used:


```r
library(tidyverse)
library(plotly)
library(lubridate)
library(sunburstR)
```

So let's get started: 

The first string contains the placement, the start number and the last name. Since the three attributes are separated by spaces, `stringr::str_split()` can be used to separate them relatively easily.

The second string is a bit more complicated. This contains the first name, the year of birth, the age group and the end time.

Since the first names are partly afflicted with double names, it is no longer possible to work with the separation by blanks. Behind the Regex coding `(?<=[a-zA-Z])\\s*(?=[0-9])` nothing else hides itself like: Extract all characters with upper and lower case letters until the next number comes.

Since the year of birth is the first number in the string, the first number can be extracted with `stringr::str_extract()`. `str_extract()` always returns the first hit. If you want to extract all matching results, you can do this with `str_extract_all()`.

The time reached represents the last eight characters in the string. Here `str_extract` helps us again.

During the grouping four patterns can be identified:

* M + a corresponding age group

* W + a corresponding age group

* as well as the respective main groups for men and women

Finally, we calculate an estimated age. Since we only have information about the year of birth, differences may occur during the year.


```r
allover_results <- womens_result %>% 
  mutate(Gender = "female") %>% 
  bind_rows(mens_result %>% 
              mutate(Gender = "male")) %>% 
  mutate(First_String = map(V1, function(x) {
    split <- unlist(str_split(x, "\\s+"))
    tibble(Place = as.numeric(split[1]),
           Startnumber = split[2],
           Surname = split[3])
  }),
  FirstName = map_chr(V2, function(x) {
    unlist(strsplit(x, 
                    split = "(?<=[a-zA-Z])\\s*(?=[0-9])", 
                    perl = TRUE))[[1]]
  }),
  Birth = as.numeric(str_extract(V2, "[0-9]+")),
  Time = lubridate::hms(as.character(str_sub(V2,-8,-1))),
  Class = map_chr(V2, function(x) {
    unlist(str_extract(x, 
                       pattern = c("M[0-9]+",
                                   "W[0-9]+",
                                   "Männer Haupt.",
                                   "Frauen Haupt."))) %>% 
      .[!is.na(.)]
  })) %>% 
  unnest() %>% 
  select(Place, Startnumber, Gender, Birth, Time, Class) %>% 
  mutate(Age_approx = 2019 - Birth,
         Time = as.numeric(as.duration(Time), "minutes"))
```

## How is the age of the runners distributed?

During the run I noticed that most runners are in middle age. With the data we can now find out exactly how age is distributed. We select a histogram that shows the age groups on the x-axis and the corresponding number on the y-axis.

It turns out that the majority of women (9) runners are between 35 and 39 years old. For men, the age group from 45 to 49 years with 22 runners is the top.


```r
allover_results %>% 
  plot_ly(alpha = 0.6,
          x = ~Age_approx,
          color = ~Gender,
          type = "histogram") %>%
  layout(barmode = "overlay")
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

This picture also shows up again when we refer to the data of the organizer and present it in a so-called sunburst diagram. In this diagram we can show several parameters such as gender and age group.


```r
allover_results %>% 
  mutate(seqs=paste(Gender,Class,sep="-")) %>% 
  group_by(seqs) %>% 
  summarise(Count = n()) %>% 
  ungroup() %>% 
  select(seqs, Count) %>% 
  sunburst(count = TRUE)
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

## How does the speed of the runners relate to the achieved placement? 

To visualize this, we plot the achieved placement on the x-axis and the end time on the y-axis. In addition, a distinction is made between women and men. 

It turns out that the time between the placements almost follows a certain linearity. In the case of men, the time difference between the individual places is smaller than in the case of women.


```r
allover_results %>% 
  plot_ly(x = ~Place,
          y = ~Time, 
          colors = "Set1") %>% 
  add_lines(color = ~Gender) %>% 
  add_markers(data = allover_results %>% 
                filter(Gender == "male",
                       Place == 62),
              x = ~Place,
              y = ~Time,
              name = "It's me :)") %>% 
  layout(yaxis = list(title = "Time [Min.]"))
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```


## At what age are the runners best?

In the following graphic we want to see in which graphic the runners are fastest. Since we are looking at the whole base we choose a box plot and create groupings for the estimated date by performing an integer division of 10. 

It turns out that both men and women in the age group 30 to 40 years represent the fastest runners. The median is about 111 minutes for women and 95 minutes for men.


```r
allover_results %>% 
  mutate(Group = (Age_approx %/% 10) * 10,
         Group = paste0(Group, "-", Group + 10)) %>% 
  plot_ly(x = ~Group, 
          y = ~Time, 
          color = ~Gender,
          type = "box") %>% 
  layout(boxmode = "group")
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

