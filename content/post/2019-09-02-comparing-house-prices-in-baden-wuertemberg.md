---
title: Comparing house prices in Baden Wuertemberg
subtitle: Part 2 -  Exploring the data
author: Florian Handke
date: '2019-09-1'
slug: comparing-house-prices-in-baden-wuertemberg-part-2
categories: []
tags: 
- dplyr
- plotly
- scraping
---

In the first post the data were collected. Now we want to get a first impression of the collected data. What is the structure of the data, how are they distributed and, most importantly, do they still need pre-processing?

In the following I will present the data in different ways. In technical jargon this is called [descriptive analytics](https://www.gartner.com/it-glossary/descriptive-analytics/). The following evaluations are only an excerpt of what is possible and do not claim to be complete.



```r
library(plotly)
library(tidyverse)
library(readr)
library(fs)
```

After I collected the data in my last post, it was saved in a local .rds file. This provides a high performance way to store data that is processed with R. With the readr package the file can be loaded via `read_rds()`.


```r
HousingData <- readr::read_rds(dir_ls("~/Documents/academic-kickstart",
                                      recurse = TRUE) %>% 
                                 path_abs() %>% 
                                 str_subset("HousingData.Rds"))
```

With the dplyr function `glimpse()`, you can quickly create an overview of individual table columns. You get the data type as well as an extract of the data in form of the first columns. If we look at the last four columns, we notice that they are all of type character and contain certain additions such as the currency or the area unit. In order to evaluate the data later, however, it is absolutely necessary to have all data in an appropriate format. Accordingly, all unnecessary attributes must be removed and then converted into a `numeric` or `double` format.


```r
glimpse(HousingData)
```

```
## Observations: 5,383
## Variables: 7
## $ County     <chr> "Alb-Donau", "Alb-Donau", "Alb-Donau", "Alb-Donau",...
## $ Title      <chr> "NEU//Schloss Dellmensingen //Denkmalschutz //Entwi...
## $ Location   <chr> "Werdensteinstraße 25, Erbach, Alb-Donau-Kreis", "L...
## $ Price      <chr> "870.000 €", "749.000 €", "519.000 €", "520.000 €",...
## $ LivingArea <chr> "426 m²", "230 m²", "197 m²", "212 m²", "166,17 m²"...
## $ Rooms      <chr> "25 ", "6,5 ", "7,5 ", "8 ", "7 ", "9 ", "10 ", "8 ...
## $ SiteArea   <chr> "3.000 m²", "1.029 m²", "875 m²", "377 m²", "761 m²...
```


```r
HousingData_clean <- HousingData %>% 
  as_tibble() %>% 
  mutate(Price = as.numeric(str_replace_all(
    str_remove_all(Price, "[€.]"), ",", ".")),
    LivingArea = as.numeric(str_replace_all(
      str_remove_all(LivingArea, "[m².]"), ",", ".")),
    Rooms = as.numeric(str_replace(Rooms, ",", ".")),
    SiteArea = as.numeric(str_replace_all(
      str_remove_all(SiteArea, "[m².]"), ",", "."))) 
```



After the cleanup, all data is now available in the desired, processable format. Additionally we can use the base-function `summary()` to get a first impression about the distribution of the numerical values.


```r
glimpse(HousingData_clean)
```

```
## Observations: 5,364
## Variables: 7
## $ County     <chr> "Alb-Donau", "Alb-Donau", "Alb-Donau", "Alb-Donau",...
## $ Title      <chr> "NEU//Schloss Dellmensingen //Denkmalschutz //Entwi...
## $ Location   <chr> "Werdensteinstraße 25, Erbach, Alb-Donau-Kreis", "L...
## $ Price      <dbl> 870000, 749000, 519000, 520000, 497000, 459000, 799...
## $ LivingArea <dbl> 426.00, 230.00, 197.00, 212.00, 166.17, 237.53, 257...
## $ Rooms      <dbl> 25.0, 6.5, 7.5, 8.0, 7.0, 9.0, 10.0, 8.0, 7.0, 6.0,...
## $ SiteArea   <dbl> 3000, 1029, 875, 377, 761, 380, 1062, 3752, 1040, 5...
```

```r
summary(HousingData_clean)
```

```
##     County             Title             Location        
##  Length:5364        Length:5364        Length:5364       
##  Class :character   Class :character   Class :character  
##  Mode  :character   Mode  :character   Mode  :character  
##                                                          
##                                                          
##                                                          
##      Price            LivingArea         Rooms           SiteArea       
##  Min.   :       1   Min.   :   0.0   Min.   : 1.000   Min.   :     0.0  
##  1st Qu.:  344000   1st Qu.: 140.0   1st Qu.: 5.000   1st Qu.:   325.8  
##  Median :  489000   Median : 180.0   Median : 6.500   Median :   573.0  
##  Mean   :  583745   Mean   : 215.7   Mean   : 7.592   Mean   :  1459.0  
##  3rd Qu.:  690000   3rd Qu.: 250.0   3rd Qu.: 9.000   3rd Qu.:   901.0  
##  Max.   :14700000   Max.   :2900.0   Max.   :58.000   Max.   :976200.0
```

We look at advertisements for the different counties in Baden Würtemberg. To determine if the number of advertisements per county varies, we group the data per county and sum them up. In the chart below we can see that the Rhine-Neckar district has the most advertisements with 417, followed by Karlsruhe (277) and Esslingen (258). The fewest advertisements are for Tübingen (82) and Emmendingen (47).


```r
HousingData_clean %>% 
  group_by(County) %>% 
  summarise(Count = n()) %>% 
  ungroup() %>% 
  plot_ly(x = ~Count,
          y = ~reorder(County, Count)) %>% 
  add_bars() %>% 
  layout(xaxis = list(title = "No of Houses"),
         yaxis = list(title = "County"))
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

To make the first summary with `summary()` even more detailed, we can create a boxplot for each attribute. For this I like to use the plotly package, because interactive SVG graphics can be created. If you want to know more about boxplots, you can find out more [here](https://towardsdatascience.com/understanding-boxplots-5e2df7bcbd51).

To create our final plot, we create a single plot (p1, p2, p3 and p4) for every attribute and combine them via subplot.


```r
p1 <- HousingData_clean %>% 
  drop_na(Price) %>% 
  plot_ly(y = ~Price, type = "box", name = "Price [€]") %>% 
  layout(showlegend = FALSE)

p2 <- HousingData_clean %>% 
  plot_ly(y = ~LivingArea, type = "box", name = "Living Area [sqm]") %>% 
  layout(showlegend = FALSE)  

p3 <- HousingData_clean %>% 
  plot_ly(y = ~SiteArea, type = "box", name = "Site Area [sqm]") %>% 
  layout(showlegend = FALSE) 

p4 <- HousingData_clean %>% 
  plot_ly(y = ~Rooms, type = "box", name = "Rooms [no]") %>% 
  layout(showlegend = FALSE) 

subplot(p1, p2, p3, p4)
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

Individual attributes can be viewed in more detail. For example, a box plot can be created for each county and displayed in comparison to the others.


```r
HousingData_clean %>% 
  drop_na(Price) %>% 
  plot_ly(x = ~Price, color = ~County, type = "box")
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

The price is often the determining factor after evaluation. However, this logic is often too brief. We want to know how the price relates to the attributes living area, site area and number of rooms.

We will also apply a linear regression to the data to see how the individual parameters behave throughout. This is achieved by the `lm()` function. The relevant values are extracted using `fitted()`.


```r
predefine_xaxis <- list(title = "Price [€]")

p1 <- HousingData_clean %>% 
  drop_na(Price) %>% 
  plot_ly(x = ~Price,
          y = ~LivingArea) %>% 
  add_markers(color = ~County,
              alpha = 0.5,
              legendgroup = ~County, 
              showlegend = T) %>% 
  add_lines(y = ~fitted(lm(LivingArea ~ Price)),
            line = list(color = '#07A4B5'),
            name = "LM", 
            legendgroup = "LM",
            showlegend = T) %>% 
  layout(xaxis = predefine_xaxis,
         yaxis = list(title = "Living Area [sqm]"))

p2 <- HousingData_clean %>% 
  drop_na(Price) %>% 
  plot_ly(x = ~Price,
          y = ~SiteArea) %>% 
  add_markers(color = ~County,
              alpha = 0.5,
              legendgroup = ~County, 
              showlegend = F) %>% 
  add_lines(y = ~fitted(lm(SiteArea ~ Price)),
            line = list(color = '#07A4B5'),
            name = "LM", 
            legendgroup = "LM",
            showlegend = F) %>% 
  layout(xaxis = predefine_xaxis,
         yaxis = list(title = "Site Area [sqm]"))

p3 <- HousingData_clean %>% 
  drop_na(Price) %>% 
  plot_ly(x = ~Price,
          y = ~Rooms) %>% 
  add_markers(color = ~County,
              alpha = 0.5,
              legendgroup = ~County, 
              showlegend = F) %>% 
  add_lines(y = ~fitted(lm(Rooms ~ Price)),
            line = list(color = '#07A4B5'),
            name = "LM", 
            legendgroup = "LM",
            showlegend = F) %>% 
  layout(xaxis = predefine_xaxis,
         yaxis = list(title = "Rooms [no]"))

subplot(p1, p2, p3, plotly_empty(), 
        nrows = 2,
        shareX = TRUE, titleY = TRUE) 
```

```
## Error in loadNamespace(name): there is no package called 'webshot'
```

This post gave us a first insight into the data. This impression is decisive when it comes to gaining added value from data in the form of reliable findings.
