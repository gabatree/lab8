Lab 08 - Text Mining/NLP
================

# Learning goals

- Use `unnest_tokens()` and `unnest_ngrams()` to extract tokens and
  ngrams from text
- Use dplyr and ggplot2 to analyze and visualize text data
- Try a theme model using `topicmodels`

# Lab description

For this lab we will be working with the medical record transcriptions
from <https://www.mtsamples.com/> available at
<https://github.com/JSC370/JSC370-2025/tree/main/data/medical_transcriptions>.

# Deliverables

1.  Questions 1-7 answered, knit to pdf or html output uploaded to
    Quercus.

2.  Render the Rmarkdown document using `github_document` and add it to
    your github site. Add link to github site in your html.

### Setup packages

You should load in `tidyverse`, (or `data.table`), `tidytext`,
`wordcloud2`, `tm`, and `topicmodels`.

## Read in the Medical Transcriptions

Loading in reference transcription samples from
<https://www.mtsamples.com/>

``` r
library(tidytext)
library(tidyverse)
```

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ dplyr     1.1.4     ✔ readr     2.1.5
    ## ✔ forcats   1.0.0     ✔ stringr   1.5.1
    ## ✔ ggplot2   3.5.1     ✔ tibble    3.2.1
    ## ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
    ## ✔ purrr     1.0.2     
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(wordcloud2)
library(tm)
```

    ## Loading required package: NLP
    ## 
    ## Attaching package: 'NLP'
    ## 
    ## The following object is masked from 'package:ggplot2':
    ## 
    ##     annotate

``` r
library(topicmodels)

mt_samples <- read_csv("https://raw.githubusercontent.com/JSC370/JSC370-2025/main/data/medical_transcriptions/mtsamples.csv")
```

    ## New names:
    ## Rows: 3682 Columns: 6
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (5): description, medical_specialty, sample_name, transcription, keywords dbl
    ## (1): ...1
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
mt_samples <- mt_samples |>
  select(description, medical_specialty, transcription)

head(mt_samples)
```

    ## # A tibble: 6 × 3
    ##   description                                    medical_specialty transcription
    ##   <chr>                                          <chr>             <chr>        
    ## 1 A 23-year-old white female presents with comp… Allergy / Immuno… "SUBJECTIVE:…
    ## 2 Consult for laparoscopic gastric bypass.       Bariatrics        "PAST MEDICA…
    ## 3 Consult for laparoscopic gastric bypass.       Bariatrics        "HISTORY OF …
    ## 4 2-D M-Mode. Doppler.                           Cardiovascular /… "2-D M-MODE:…
    ## 5 2-D Echocardiogram                             Cardiovascular /… "1.  The lef…
    ## 6 Morbid obesity.  Laparoscopic antecolic anteg… Bariatrics        "PREOPERATIV…

------------------------------------------------------------------------

## Question 1: What specialties do we have?

We can use `count()` from `dplyr` to figure out how many different
medical specialties are in the data. Are these categories related?
overlapping? evenly distributed? Make a bar plot.

The distribution of medical specialties in the dataset is highly uneven,
with Surgery having the most transcriptions, followed by Orthopedic,
Radiology, and General Medicine, while many other specialties have
significantly fewer entries. Some categories may overlap, such as
General Medicine potentially including cases from multiple specialties,
and SOAP/Chart/Progress Notes covering diverse medical fields.

``` r
mt_samples |>
  count(medical_specialty, sort = TRUE) |>
  ggplot(aes(x = n, y = fct_reorder(medical_specialty, n))) +
  geom_col(fill = "lightblue") +
  theme_minimal()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

------------------------------------------------------------------------

## Question 2: Tokenize

- Tokenize the the words in the `transcription` column
- Count the number of times each token appears
- Visualize the top 20 most frequent words with a bar plot
- Create a word cloud of the top 20 most frequent words

### Explain what we see from this result. Does it makes sense? What insights (if any) do we get?

The chart shows that the most frequent words in the medical
transcription dataset are common stopwords like “the,” “and,” “was,” and
“of,” which do not provide meaningful insights. To extract useful
medical information, we should remove stopwords and focus on
domain-specific terms or phrases.

``` r
tokens <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  group_by(word) |>
  summarize(word_frequency = n()) |>
  arrange(across(word_frequency, desc)) |>
  head(20)
tokens
```

    ## # A tibble: 20 × 2
    ##    word    word_frequency
    ##    <chr>            <int>
    ##  1 the             116095
    ##  2 and              60381
    ##  3 was              58047
    ##  4 of               42147
    ##  5 to               36842
    ##  6 a                31120
    ##  7 with             26462
    ##  8 in               23955
    ##  9 is               15842
    ## 10 patient          14971
    ## 11 were             12712
    ## 12 for              11444
    ## 13 no               10835
    ## 14 she              10713
    ## 15 then             10563
    ## 16 this             10307
    ## 17 on               10105
    ## 18 he                9717
    ## 19 at                9677
    ## 20 right             8540

``` r
tokens |>
  ggplot(aes(fct_reorder(word,word_frequency),word_frequency)) +
  geom_bar(stat="identity",fill = "lightblue")+
  coord_flip()+
  theme_bw()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
tokens |> 
  count(word, sort=TRUE) |>
  wordcloud2(size = 0.5,color = "random-light",backgroundColor = "lightblue")
```

------------------------------------------------------------------------

## Question 3: Stopwords

- Redo Question 2 but remove stopwords
- Check `stopwords()` library and `stop_words` in `tidytext`
- Use regex to remove numbers as well
- Try customizing your stopwords list to include 3-4 additional words
  that do not appear informative

### What do we see when you remove stopwords and then when you filter further? Does it give us a better idea of what the text is about?

When stopwords are removed, the most frequent words become more
domain-specific, revealing terms like “pain,” “surgery,” “treatment,”
and “patient”, which are more relevant to medical transcriptions.

``` r
head(stopwords("english"))
```

    ## [1] "i"      "me"     "my"     "myself" "we"     "our"

``` r
length(stopwords("english"))
```

    ## [1] 174

``` r
head(stop_words)
```

    ## # A tibble: 6 × 2
    ##   word      lexicon
    ##   <chr>     <chr>  
    ## 1 a         SMART  
    ## 2 a's       SMART  
    ## 3 able      SMART  
    ## 4 about     SMART  
    ## 5 above     SMART  
    ## 6 according SMART

``` r
tokens2 <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  anti_join(stop_words, by = "word") |>
  filter(!str_detect(word, "^[0-9]+$")) |> 
  filter(!word %in% c("mm","mg","noted")) |>
  group_by(word) |>
  summarize(word_frequency = n()) |>
  arrange(across(word_frequency, desc)) |>
  head(20)
tokens2
```

    ## # A tibble: 20 × 2
    ##    word          word_frequency
    ##    <chr>                  <int>
    ##  1 patient                14971
    ##  2 left                    8026
    ##  3 procedure               6214
    ##  4 history                 5044
    ##  5 normal                  4862
    ##  6 pain                    3798
    ##  7 anesthesia              3200
    ##  8 performed               3180
    ##  9 incision                3166
    ## 10 time                    2979
    ## 11 removed                 2948
    ## 12 skin                    2928
    ## 13 diagnosis               2665
    ## 14 blood                   2602
    ## 15 anterior                2367
    ## 16 position                2242
    ## 17 fashion                 2111
    ## 18 vicryl                  2038
    ## 19 closed                  2020
    ## 20 postoperative           1928

``` r
tokens2 |>
  ggplot(aes(fct_reorder(word,word_frequency),word_frequency)) +
  geom_bar(stat="identity",fill = "lightblue")+
  coord_flip()+
  theme_bw()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

``` r
tokens2 |> 
  count(word, sort=TRUE) |>
  wordcloud2(size = 0.4,color = "random-light",backgroundColor = "lightblue")
```

<div class="wordcloud2 html-widget html-fill-item" id="htmlwidget-1fd2c399dedce54be9f7" style="width:672px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-1fd2c399dedce54be9f7">{"x":{"word":["anesthesia","anterior","blood","closed","diagnosis","fashion","history","incision","left","normal","pain","patient","performed","position","postoperative","procedure","removed","skin","time","vicryl"],"freq":[1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],"fontFamily":"Segoe UI","fontWeight":"bold","color":"random-light","minSize":0,"weightFactor":72,"backgroundColor":"lightblue","gridSize":0,"minRotation":-0.7853981633974483,"maxRotation":0.7853981633974483,"shuffle":true,"rotateRatio":0.4,"shape":"circle","ellipticity":0.65,"figBase64":null,"hover":null},"evals":[],"jsHooks":{"render":[{"code":"function(el,x){\n                        console.log(123);\n                        if(!iii){\n                          window.location.reload();\n                          iii = False;\n\n                        }\n  }","data":null}]}}</script>

------------------------------------------------------------------------

## Question 4: ngrams

Repeat question 2, but this time tokenize into bi-grams. How does the
result change if you look at tri-grams? Note we need to remove stopwords
a little differently. You don’t need to recreate the wordclouds.

The bi-gram analysis provides clearer medical context, revealing key
terms like “blood pressure,” “medical history,” and “postoperative
diagnosis”, which are more informative than single words. This helps
identify common themes in medical documentation, and further refining
with tri-grams or categorization could enhance insight extraction.

``` r
stop_words2 <- c("en", "mm", "mg", "count", stop_words$word)
sw_start <- paste0("^", paste(stop_words2, collapse=" |^"), "$")
sw_end <- paste0("", paste(stop_words2, collapse="$| "), "$")

tokens_bigram <- mt_samples |>
  select(transcription) |>
  unnest_tokens(ngram, transcription, token = "ngrams", n = 2) |>
  filter(!grepl(sw_start, ngram, ignore.case = TRUE))|>
  filter(!grepl(sw_end, ngram, ignore.case = TRUE))|>
  filter(!grepl("[[:digit:]]+", ngram))|>
  group_by(ngram) %>% 
  summarize(word_frequency = n()) %>% 
  arrange(across(word_frequency, desc)) %>% 
  head(20)

tokens_bigram %>% 
  ggplot(aes(ngram, word_frequency)) +
  geom_col(fill="dodgerblue") +
  coord_flip() +
  theme_bw()
```

## ![](08lab-text-mining_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

## Question 5: Examining words

Using the results from the bigram, pick a word and count the words that
appear before and after it, and create a plot of the top 20.

``` r
library(stringr)
# e.g. patient, blood, preoperative...
tokens_bigram |>
  filter(str_detect(ngram, regex("\\sblood$|^blood\\s"))) %>% 
    mutate(word = str_remove(ngram, "blood"),
         word = str_remove_all(word, " ")) |>
  group_by(word) %>% 
  head(20) %>% 
  ggplot(aes(reorder(word, word_frequency), word_frequency)) +
  geom_col(fill="dodgerblue") +
  theme_bw()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

------------------------------------------------------------------------

## Question 6: Words by Specialties

Which words are most used in each of the specialties? You can use
`group_by()` and `top_n()` from `dplyr` to have the calculations be done
within each specialty. Remember to remove stopwords. How about the 5
most used words?

``` r
mt_samples |>
   unnest_tokens(word, transcription) |>
   anti_join(stop_words, by = "word") |>
  filter(!str_detect(word, "^[0-9]+$")) |> 
  filter(!word %in% c("mm","mg","noted")) |>
   group_by(medical_specialty) |>
   count(word,sort = TRUE)|>
  top_n(5,n)
```

    ## # A tibble: 172 × 3
    ## # Groups:   medical_specialty [30]
    ##    medical_specialty word           n
    ##    <chr>             <chr>      <int>
    ##  1 Surgery           patient     4855
    ##  2 Surgery           left        3263
    ##  3 Surgery           procedure   3243
    ##  4 Orthopedic        patient     1711
    ##  5 Surgery           anesthesia  1687
    ##  6 Surgery           incision    1641
    ##  7 General Medicine  patient     1356
    ##  8 General Medicine  history     1027
    ##  9 Orthopedic        left         998
    ## 10 Gastroenterology  patient      835
    ## # ℹ 162 more rows

## Question 7: Topic Models

See if there are any themes in the data by using a topic model (LDA).

- you first need to create a document term matrix
- then you can try the LDA function in `topicmodels`. Try different k
  values.
- create a facet plot of the results from the LDA (see code from
  lecture)

``` r
transcripts_dtm <- mt_samples |>
  select(transcription) |>
  unnest_tokens(word, transcription) |>
  anti_join(stop_words, by = "word") |>
  filter(!str_detect(word, "^[0-9]+$")) |> #[[:digit:]]+
  filter(!word %in% c("mm", "mg", "noted")) |>
  DocumentTermMatrix()

transcripts_dtm <- as.matrix(transcripts_dtm)   

transcripts_lda1 <- LDA(transcripts_dtm, k = 5,
                       control = list(seed = 1234))

transcripts_lda2 <- LDA(transcripts_dtm, k = 3,
                       control = list(seed = 1234))

transcripts_top_terms1 <- 
  tidy(transcripts_lda1, matrix = "beta") |>
  filter(!str_detect(term, "^[0-9]+$")) |>
  group_by(topic) |>
  slice_max(beta, n = 10) |>
  ungroup() |>
  arrange(topic, -beta)

transcripts_top_terms2 <- 
  tidy(transcripts_lda2, matrix = "beta") |>
  filter(!str_detect(term, "^[0-9]+$")) |>
  group_by(topic) |>
  slice_max(beta, n = 10) |>
  ungroup() |>
  arrange(topic, -beta)

transcripts_top_terms1 |>
  mutate(term = reorder_within(term, beta, topic)) |>
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~topic, scales = "free") +
  scale_y_reordered() +
  theme_bw()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

``` r
transcripts_top_terms2 |>
  mutate(term = reorder_within(term, beta, topic)) |>
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~topic, scales = "free") +
  scale_y_reordered() +
  theme_bw()
```

![](08lab-text-mining_files/figure-gfm/unnamed-chunk-9-2.png)<!-- -->
