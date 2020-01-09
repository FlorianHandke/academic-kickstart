---
title: Extracting the essential
author: Florian Handke
date: '2019-07-27'
slug: extracting-the-essential
categories: []
tags:
  - rvest
  - udpipe
  - textrank
  - Pagerank
  - Natural language processing
subtitle: ''
summary: ''
authors: []
lastmod: '2019-07-27T16:17:59+02:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Reading large texts takes time. Sometimes it would be useful to extract essential sentences of a text to get a first impression of the whole. Therefore we will scrap a text from the web, do some string manipulation, tokenize it and finally extract the most important sentences by using Google Pagerank algorithm. 

To do so I will use the following R packages

 * `rvest`
 
 * `stringr`
 
 * `udpipe`
 
 * `textrank`
 
In our example, we will use a text of the [The Bureau of investegative journalism](https://www.thebureauinvestigates.com/) by [Madlen Davies](https://www.thebureauinvestigates.com/profile/madlendavies) and [Ben Stockton](https://www.thebureauinvestigates.com/profile/Benstockton) from July 22 2019. The article describes the ban of a antibioticum in Indian farms to fatten up animals. Fatten up animals is - according to the WHO - a major cause of the world's growing antibiotic resistance crisis.

You can find the full article [here](https://www.thebureauinvestigates.com/stories/2019-07-22/india-bans-use-of-last-hope-antibiotic-colistin-on-farms).

To scrape the text from the website we will use the [rvest](https://cran.r-project.org/web/packages/rvest/rvest.pdf) package which makes it easy to get all the data we want. It also provides assistance to use pipes from the [magrittr](https://magrittr.tidyverse.org/) package which is always a good choice. :)


```r
library(rvest)
```

Scraping the data we simply define the relevant URL and a node which indicates a corresponding CSS selector. Because we only want the text of our article, we choose `p` as our relevant node. The [p](https://html.com/tags/p/#ixzz5utCwr0gg) element is used to identify blocks of paragraph text in HTML.

There is a wide variety of nodes we can define but do not need in this context. For exmaple we could scrape tables, rating, pictures and so on. 


```r
url <- 'https://www.thebureauinvestigates.com/stories/2019-07-22/india-bans-use-of-last-hope-antibiotic-colistin-on-farms'

webpage <- read_html(url)

text <- webpage %>% 
  html_nodes("p") %>% 
  html_text() 
```

In the next step we will do a little string manipulation to

 * split our text in sentences (`str_split`)
 
 * get rid of some symbols we do not wanna have (`str_trim`) and whitespace at the begin and ending (`str_trim`)
 
 

```r
library(stringr)
```


```r
text <- unlist(strsplit(text, "\\. ")) %>% 
  str_replace_all(pattern = "\n", replacement = " ") %>%
  str_replace_all(pattern = "[\\^]", replacement = " ") %>%
  str_replace_all(pattern = "\"", replacement = " ") %>%
  str_replace_all(pattern = "\\s+", replacement = " ") %>%
  str_trim(side = "both") 
```




Our text has now a total number of **776 words**.

Next thing we wanna do is tokenizing and tagging our text. Therefore we use the [udpipe](https://cran.r-project.org/web/packages/udpipe/index.html) package.


```r
library(udpipe)
```

To do so we need a udpipe-model in english language.


```r
udmodel <- udpipe_download_model("english")
udmodel <- udpipe_load_model(udmodel$file_model)
```

Once we loaded the model we can annotate our sentences by calling `udpipe_annotate()` and transforming it to a dataframe.


```r
df_text <- udpipe_annotate(udmodel, text) %>% 
  as.data.frame(text)

head(df_text)
```

```
##   doc_id paragraph_id sentence_id                        sentence token_id
## 1   doc1            1           1 We tell the stories that matter        1
## 2   doc1            1           1 We tell the stories that matter        2
## 3   doc1            1           1 We tell the stories that matter        3
## 4   doc1            1           1 We tell the stories that matter        4
## 5   doc1            1           1 We tell the stories that matter        5
## 6   doc1            1           1 We tell the stories that matter        6
##     token  lemma upos xpos                                      feats
## 1      We     we PRON  PRP Case=Nom|Number=Plur|Person=1|PronType=Prs
## 2    tell   tell VERB  VBP           Mood=Ind|Tense=Pres|VerbForm=Fin
## 3     the    the  DET   DT                  Definite=Def|PronType=Art
## 4 stories  story NOUN  NNS                                Number=Plur
## 5    that   that PRON  WDT                               PronType=Rel
## 6  matter matter  ADV   RB                                       <NA>
##   head_token_id   dep_rel deps            misc
## 1             2     nsubj <NA>            <NA>
## 2             0      root <NA>            <NA>
## 3             4       det <NA>            <NA>
## 4             2       obj <NA>            <NA>
## 5             6     nsubj <NA>            <NA>
## 6             4 acl:relcl <NA> SpacesAfter=\\n
```

To find the most important senteces we will use a graph based ranking model by using an unsupervised method. 

Textrank constructs a graph where "the verticles of a the graph represent each sentence in our text and the edges between sentences are based on content overlap, namely by calculating the number of words that two sentences have in common."

Textrank then uses [PageRank](https://www.cs.princeton.edu/~chazelle/courses/BIB/pagerank.htm) to identify the most important sentences of a text. PageRank is also used by Google Search to rank web pages.

If you are interested in more details please read the [paper](https://web.eecs.umich.edu/~mihalcea/papers/mihalcea.emnlp04.pdf).


```r
library(textrank)
```

Textrank needs a dataframe with sentences and a dataframe with words which are part of these sentences.

Here we only take nouns and adjectives for finding overlaps between sentences.


```r
df_text$textrank_id <- unique_identifier(df_text, c("doc_id", "paragraph_id", "sentence_id"))
sentences <- unique(df_text[, c("textrank_id", "sentence")])
terminology <- subset(df_text, upos %in% c("NOUN", "ADJ"))
terminology <- terminology[, c("textrank_id", "lemma")]
head(terminology)
```

```
##    textrank_id   lemma
## 4            1   story
## 9           12  defend
## 10          12 quality
## 13          12   spark
## 14          12  change
## 34          37   story
```

By applying our sentences and terminology into `textrank_sentences()` they will be applied to Google's Pagerank.


```r
textrank <- textrank_sentences(data = sentences, terminology = terminology)
```

We can now extract the top n most relevant sentences where n defines the number of sentences which should be mentioned.


```r
important_sentences <- summary(textrank, 
                               n = 4,
                               keep.sentence.order = TRUE)
cat(important_sentences, sep = "\n")
```

```
## The Indian government has banned the use of a “last hope” antibiotic on farms to try to halt the spread of some of the world’s most deadly superbugs, after a Bureau investigation revealed it was being widely used to fatten livestock.
## The use of antibiotics to fatten up animals — known as “growth promotion” — is a major cause of the world's growing antibiotic resistance crisis
## He said the ban indicates that “the Indian government is convinced that colistin is a last resort antibiotic, colistin resistance is increasing in clinical practice and colistin is extensively used in poultry and aqua farming as a growth promoting agent” and such practice should stop.
## The discovery of a colistin-resistant gene that can pass between bacteria, conferring resistance to bugs never exposed to the drug, sent shockwaves through the medical world in 2015
```

We redouced our text from **776 to 139** words by using natural language processing in combination with Google Pagerank. 

Do you understand the core message of the text by reading only the summary? 
