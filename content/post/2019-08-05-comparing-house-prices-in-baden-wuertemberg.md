---
title: Comparing house prices in Baden Wuertemberg
subtitle: Part 1 -  Downloading the data
author: Florian Handke
date: '2019-09-1'
slug: comparing-house-prices-in-baden-wuertemberg
categories: []
tags:
- rvest
- purrr
- scraping
---

The price of houses is widely discussed these days in germany:

Real estate prices have risen nationwide in recent years. This is also shown by the present house price index of the Federal Statistical Office, which, starting from 2015 (index = 100), was around 116.3 points in 2018. Thus, prices have increased by 16.3 percent compared to the base year 2015. Nevertheless, there are regional differences in the development of real estate prices. [statista](https://de.statista.com/statistik/daten/studie/70265/umfrage/haeuserpreisindex-in-deutschland-seit-2000/)

Houses are becoming more expensive almost everywhere in Europe - especially in Germany. This was calculated by Eurostat, the European Statistics Office. While in the euro zone prices for residential buildings rose by 4.5 percent in the first quarter, the figure in Germany was as high as 5.3 percent. In the EU as a whole, an average of 4.7 percent more had to be paid for a house than in the first quarter of 2017.
[handelsblatt](https://www.handelsblatt.com/finanzen/immobilien/europaeischer-vergleich-hauspreise-in-deutschland-ueberdurchschnittlich-stark-gestiegen/22788824.html?ticket=ST-6190196-5Wioef2UO3o6HZwheSC6-ap1)

Since I live in Baden-Wuerttemberg, I am mainly interested in the regional house price:

+ What is the gradient of the city/country?

+ Where are the houses cheapest per square meter?

+ Where are the most expensive houses?

I will shed light on all these factors in the coming blog posts. For this I want to examine house prices of a real estate portal.

In my first post I will create the data basis for further analyses. For this purpose, I will obtain the data in R by means of webscraping. As already in my previous posts the R parcel rvest will be used.

In further posts I will then examine the data further.

To scrapn the data, we will use a few packages with useful functions:

* **rvest** to scrape the data from the website

* **tidyverse** which includes magrittr (piping), stringr (string manipulation), dplyr (data wraggling), purrr (calling functions)...

* **glue** to glue strings :)


```r
library(rvest)
library(tidyverse)
library(glue)
```

## Helper Functions

The package purrr allows us to call functions within a dataframe and pass arguments to them. In this case we do not really need it, but i love working with it...

To get the data I built a loop to scrape the data and a helper function to get the information attributes. 

Due to inconsistent data we need a helper function (SplitIt). This function checks individual arguments and merges them into a tibble. Missing arguments are replaced by an NA.






```r
## Helper function
## Price, LivingArea, Rooms, and SiteArea are no necessary arguments. therefore we need to check if they are existant.

SplitIt <- function(string) {
  
  ## Splitting the string into the different arguments
  
  string_split <- unlist(str_split(string, "(?<=Kaufpreis|Wohnflaeche|Zi.)"))
  
  ## Combine everything to a tibble. If the argument is missing fill a NA. Afterwards replace the name with ""
  
  tibble(Price = ifelse(any(str_detect(string_split, "Kaufpreis")),
                        string_split[str_detect(string_split, "Kaufpreis")],
                        NA) %>% str_replace("Kaufpreis", ""),
         LivingArea = ifelse(any(str_detect(string_split, "Wohnflaeche")),
                             string_split[str_detect(string_split, "Wohnflaeche")],
                             NA) %>% str_replace("Wohnflaeche", ""),
         Rooms = ifelse(any(str_detect(string_split, "Zi.")),
                        first(string_split[str_detect(string_split, "Zi.")]),
                        NA) %>% str_replace("Zi.", ""),
         SiteArea = ifelse(any(str_detect(string_split, "Grundstueck")),
                           string_split[str_detect(string_split, "Grundstueck")],
                           NA) %>% str_replace("Grundstueck", ""))
  
}
```



## Executing the for loop

The data is saved as a tibble for further processing. Then it can be passed to the helper function (SplitIt) via purrr::map(). The helper function then returns data as tibble. 



```r
## To get to our landing page we simulate our first page argument with 1

page <- 1

## glue the url to get pagination

startpage <- read_html(glue(url))

pagination <- startpage %>% 
  html_nodes(".select.font-standard > option") %>% 
  html_text() %>% 
  as.numeric() %>% 
  .[!is.na(.)]

HousingData <- tibble()

## Looping over all pages

for (i in seq_along(pagination)) {
  
  page <- i
  
  ## glue the url for a single page
  
  singlepage <- read_html(glue(url))
  
  ## Predefining the relevant node
  
  first_lvl_node <- singlepage %>% 
    html_nodes(paste0(".listings-content-area > .react > ",
                      "div > div > ul > li > div > article > ",
                      "div:nth-of-type(1) > div:nth-of-type(2)"))
  
  ## Creating the Result for one page
  
  PageResult <- 
    tibble(Title = first_lvl_node %>% 
             html_nodes(paste0("div > a > h5")) %>% 
             html_text(),
           Location = first_lvl_node %>% 
             html_nodes(paste0("div > div:nth-of-type(2) > div > button")) %>% 
             html_text(),
           RestData = first_lvl_node %>% 
             html_nodes("div > div:nth-of-type(3) > div > div:nth-of-type(1)") %>% 
             html_text()) %>% 
    mutate(RestData = str_replace(RestData,"Wohnfläche", "Wohnflaeche"), ## stringr does not recognize umlauts
           RestData = str_replace(RestData,"Grundstück", "Grundstueck"),
           SplitString = map(RestData, SplitIt)) %>% 
    unnest() %>% 
    select(-RestData)
  
  ## Combing all results
  
  HousingData <- HousingData %>% 
    bind_rows(PageResult)
  
}
```
And our rsult looks like this. It conatains 6025 rows which equals the number of houses hosted on the portal


```r
head(HousingData)
```

```
## # A tibble: 6 x 6
##   Title                 Location           Price  LivingArea Rooms SiteArea
##   <chr>                 <chr>              <chr>  <chr>      <chr> <chr>   
## 1 NEUSchönes 3-FH für ~ Ketsch, Rhein-Nec~ 699.0~ 276 m²     "10 " 480 m²  
## 2 Schöne Doppelhaushäl~ Kehl, Ortenaukreis 399.0~ 110 m²     "4 "  299 m²  
## 3 NEUIhr neues Wohnglü~ Sindelfingen, Böb~ 520.0~ 175 m²     "7 "  238 m²  
## 4 NEUElegantes und Mod~ Untertürkheim, St~ 998.0~ 140 m²     "5,5~ 397 m²  
## 5 NEUNeu bauen oder sa~ Bad Cannstatt, St~ 520.0~ 105 m²     "6 "  307 m²  
## 6 NEUKernen im Remstal~ Hölderlinstraße 7~ 1.190~ 287,86 m²  "12,~ 785 m²
```



The data looks great and I am curious to see what information and insights we can generate from it.

In my next post I will examine the data more closely and perform a desriptive analysis. 
