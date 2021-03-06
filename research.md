# Machine Learning for Human Activity Recognition
Ion Scerbatiuc  
October 22, 2016  



## Synopsis

Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement – a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. 

One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. In this research paper we will use sensory data from various accelerometers to predict how well the participants performed on the given physical excercise.

Six young health participants were asked to perform one set of 10 repetitions of the Unilateral Dumbbell Biceps Curl in five different fashions: exactly according to the specification (Class A), throwing the elbows to the front (Class B), lifting the dumbbell only halfway (Class C), lowering the dumbbell only halfway (Class D) and throwing the hips to the front (Class E).

Read more about the experiment and the available data sources here: http://groupware.les.inf.puc-rio.br/har#ixzz4NqVb46KZ

## Prerequisites

In order to be able to reproduce this research you'll need to install the following R packages: caret, randomForest and rpart. We also create a function `downloadData` to aid us in the data retrieval step.


```r
install.packages(c('caret', 'rpart', 'randomForest'))

downloadData <- function(url) {
    # Download the data from the provided URL and return the path to the
    # downloaded file.
    if (!file.exists("data")) {
        dir.create("data")
    }

    filename <- basename(url)
    destfile = paste(c("data", filename), collapse="/")
    if (!file.exists(destfile)) {
        download.file(url, destfile = destfile, method = "curl")
    }

    destfile
}
```

## Getting and cleaning the data

First, let's load the needed dependencies


```r
library(caret)
```

Next, let's fetch and load the training and testing datasets.


```r
trainingDataFile <- downloadData('https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv')
testingDataFile <- downloadData('https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv')

trainingData <- read.csv(trainingDataFile, na.strings=c("NA",""))
testingData <- read.csv(testingDataFile, na.strings=c("NA",""))
```

By using `summary(trainingData)` we conclude that the first 7 columns of the datasets are informational and don't have any predictive value. There are also a lot of columns with missing data. We'll exclude all these columns from both datasets. At the end, we retain 52 predictors for the `classe` variable we're trying to predict.


```r
trainingData <- trainingData[, -c(1:7)]
testingData <- testingData[, -c(1:7)]

valid.columns <- colSums(is.na(trainingData)) == 0
trainingData <- trainingData[, valid.columns]
testingData <- testingData[, valid.columns]
```

## Modeling considerations

The first step in building the classifier is to split the training set into two subsets: training and validation. We will train our models on the training subset and will use the validation set to compare the performance of the models and to estimate the out-of-sample error of the final model. 

Since the variable we're trying to predict (`classe`) is a factor variable (takes discrete values), we need to build a classification model (versus a regression model that is used to predict a continuous variable). In case of classifiers, the out-of-sample error on the testing set can be estimated by `1 - accuracy` of the model on the validation set.


```r
set.seed(12321)

inTrain <- createDataPartition(y=trainingData$classe, p=0.75, list=FALSE)
trainingSet <- trainingData[inTrain, ]
validationSet <- trainingData[-inTrain, ]
```

Decision trees and random forests are known as good models for classification problems. By the nature of the algorithm (sub-sampling and selecting by vote the best sub-trees), they also are good at picking the most predictive features. Therefore we will use all the remaining features to build the models and let the models determine the best features. We will train a decision tree model (using the `rpart` library) and a random forest model (using the `randomForest` library) and will compare their accuracies. The model with the best accuracy will be choosen.

## Training and evaluating the classifiers

In this step we will train the two classifiers. This operation can take a long time, so be prepared to wait until the algorithms finish to run and you get the final models.


```r
set.seed(23432)

fitRpart <- train(classe ~ ., data=trainingSet, method="rpart", tuneLength=20)
fitRF <- train(classe ~ ., data=trainingSet, method="rf")
```

Next, let's compare the accuracies of the two models


```r
confusionMatrix(predict(fitRpart, validationSet), validationSet$classe)$overall['Accuracy']
```

```
##  Accuracy 
## 0.7999592
```


```r
confusionMatrix(predict(fitRF, validationSet), validationSet$classe)$overall['Accuracy']
```

```
##  Accuracy 
## 0.9934747
```

From the results above, we conclude that the random forest classifier has a better accuracy (0.993) than the decision tree classifier (0.799). We therefore choose to use the former to evaluate the testing data set. The estimated out-of-sample error of the classifier is `1 - 0.993 = 0.007`, which means that we will misclassify only about 7 observations in a thousand.

For the random forest that we chose, let's visualize the most important variables that the model identified. For that we can use the `varImpPlot` function on the final model.


```r
varImpPlot(fitRF$finalModel, n.var=20, main="20 most important variables (from 52)")
```

![](research_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

From the plot above we can see that the yaw, pitch and roll measurements on the belt are in the top 5 most important variables, which is not surprising given on the type of physical exercise (the subject need to not move the belt a lot when performing the exercise, so any changes in the readings on the belt are expected to be highly predictive).

## Evaluating the final model on the testing dataset

Finally, let's use our model to predict the classes of the testing observations. Since the out-of-sample error is estimated to be `0.007`, the probability of misclassifying any observation in this dataset is `dbinom(1, size=20, prob=0.007) = 0.123`. That is, there is only a `12%` chance that one of the observations will be misclassified.


```r
predict(fitRF, testingData)
```

```
##  [1] B A B A A E D B A A B C B A E E A B B B
## Levels: A B C D E
```

Running the above results through the automatically graded quiz, yields a 100% accuracy on this testing set of 20 observations.
