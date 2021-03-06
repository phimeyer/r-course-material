Supervised Text Classification
================
Wouter van Atteveldt & Kasper Welbers
2020-10

In supervised text classification, we train a statistical model on the
*features* of our data (e.g. the word frequencies) to predict the
*class* of our texts (e.g. the sentiment).

For this example, we will use functions from the `quanteda.textmodels`
package. Normally, the go-to package for machine learning is `caret`,
but unfortunately that package does not deal with sparse matrices well,
making it less attractive for text analysis purposes. The good thing is
that if you are used to `quanteda` you will find it very easy to use
their textmodels as well.

# Data

For this example, we will use Amazon reviews on automotive products.
These reviews have the benefit of being relatively straightforward and
explicit in their expressed sentiment (e.g. compared to parliamentary
speeches), and there is a large amount of existing reviews that can be
downloaded.

We use the reviews from [Amazon Review
Dataset](https://nijianmo.github.io/amazon/index.html) (scroll down to
the ‘small’ data sets which are freely available). These reviews are
stored in gzipped json-lines format, meaning it is a compressed file in
which each line is a json document. This sounds complicated, but you can
directly read this into an R data frame using the `jsonlite::stream_in`
function on the url using the `gzcon` function to decompress.

For this example, we pick the beauty category, mostly because it is
relatively small. If you select a category with more data, it will take
longer to download and run the models, but results might well be
better.

``` r
library(tidyverse)
```

``` r
review_url = "http://deepyeti.ucsd.edu/jianmo/amazon/categoryFilesSmall/Luxury_Beauty_5.json.gz"
reviews = jsonlite::stream_in(gzcon(url(review_url))) %>% 
   as_tibble() %>% select(reviewerID, asin, overall, summary, reviewText)
reviews
```

In this file, `reviewID` identifies the user that placed the reivew,
`asin` identifies the reviewed product, `overall` is the amount of
stars, and `summary` and `reviewText` are the review text as entered by
the user. Taken together, `reviewID` and `asin` uniquely identify a row.

Before proceeding to the text classification, we will compute a binary
target class (five-star or not) and create a text variable combining the
summary and review text:

``` r
reviews = reviews %>% mutate(fivestar = overall == 5,
   text = str_c(str_replace_na(summary),  str_replace_na(reviewText), sep=" "))
```

## Splitting into training and test data

Before we can train a model, we need to split the model into training
and text data. We do this with regular R and tidyverse functions, in
this case we sample from the row indices and use `slice` to select the
appropriate rows (using the negative selection for the test set to
select everything except for the training set):

``` r
trainset = sample(nrow(reviews), size=round(nrow(reviews) * 0.8))
reviews_train = reviews %>% slice(trainset)
reviews_test = reviews %>% slice(-trainset)
```

## Creating the DFM and ML Model

First, we create a document feature model of the training data:

``` r
library(quanteda)
dfm_train = reviews_train %>%  corpus() %>% dfm(stem=T) %>% dfm_trim(min_docfreq = 10)
```

Now, we can train a text model such as naive bayes:

``` r
library(quanteda.textmodels)
nbmodel = textmodel_nb(dfm_train, dfm_train$fivestar)
summary(nbmodel)
```

Let’s test it on the training data set (note, this should always yield
good results unless something went wrong)

``` r
predictions = predict(nbmodel, dfm_train)
mean(predictions == dfm_train$fivestar)
```

## Validating on the test data

To use the model on new data, we need to make sure that the columns of
the train and test dataset agree. For this, we can use the `dfm_match`
function, which makes sure that the test dfm uses the columns from the
train
dfm:

``` r
dfm_test = reviews_test %>%  corpus() %>% dfm(stem=T) %>% dfm_match(features = featnames(dfm_train))
colnames(dfm_test) %>%  head()
colnames(dfm_train) %>%  head()
```

Now, we can use the predict function as above:

``` r
predictions = predict(nbmodel, dfm_test)
mean(predictions == dfm_test$fivestar)
```

So, the result is lower, but not that much lower, which is good. Next,
let’s have a look at some more statistics, for which we use the `caret`
package:

``` r
library(caret)
confusionMatrix(table(predictions, dfm_test$fivestar), mode = "prec_recall", positive="TRUE")
```

This results shows us the confusion matrix and a number of performance
statistics, including precision (if it predicted 5 stars, was it
correct), recall (out of all 5-star reviews, how many did it predict),
and F1 (harmonic mean of precision and recall)

# Where to go next?

The code above gives a good template to get started with supervised text
analysis. First, you can try using different data sets and different
models, e.g. the support vector machine `textmodel_svm` Next, you can
experiment with different features, e.g. by trying n-grams,
lemmatization, etc. Then, you can use crossvalidation rather than a
single train-test split, especially to find the best parameter settings
for models that have hyperparameters (such as the C and epsilon
parameters of the SVM). Finally, you can experiment with how much
training data is required to produce a good model by running the model
with random subsamples of the training data, producing a graph called a
*learning curve*.
