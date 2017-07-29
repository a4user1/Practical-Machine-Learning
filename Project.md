---
title: "Prediction Assignment"
author: "Carlos Barco"
date: "7/27/2017"
output:
  html_document: default
  pdf_document: default
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(knitr)
```

#Introduction
Data collection through self-monitoring and self-sensing combines wearable sensors (e.g.Electroencephalography, Electrocardiography) and [wearable computing](https://en.wikipedia.org/wiki/Wearable_computer) (smartwatchs, heart rate monitors, etc.) [See Quantified self](https://en.wikipedia.org/wiki/Quantified_Self).  
Using wereable computers like known fitness accessories is now possible to collect a large amount of data about fitness personal activity in an inexpensively way. This trackers devices are part of a “life logging” movement, and a general characteristic of the information collected is that is known how much a particular activity was did it, but rarely quantified how well they did it.  
In this project, the goal is to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website: http://groupware.les.inf.puc-rio.br/har (see the section on the Weight Lifting Exercise Dataset).  

#Executive Resume
The description of the assignment contains the following information on the dataset:  
In this project, your goal will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways.  
The goal is  
to predict the manner in which they did the exercise.  

##Building the model

###Reproducibility
####Preparing the data and R packages
The following Libraries were used for this project, so you should install and load them in your own working environment.
```{r}
library(caret)
library(rpart)
library(rpart.plot)
library(randomForest)
library(corrplot)
library(RColorBrewer)
```

```{r}
getRversion()
```

I choose randomForest (as a supervised learning algorithm), because of their known utility in classification problems, and gives an acceptable level of performance.

####Getting Data
```{r}
trainUrl <-"https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
testUrl <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
trainFile <- "/Users/carlosbarco/R/pml-training.csv"
testFile  <- "/Users/carlosbarco/R/pml-testing.csv"
if (!file.exists("./data")) {
  dir.create("./data")
}
if (!file.exists(trainFile)) {
  download.file(trainUrl, destfile = trainFile, method = "curl")
}
if (!file.exists(testFile)) {
  download.file(testUrl, destfile = testFile, method = "curl")
}
```

####Reading Data
```{r}
trainRaw <- read.csv(trainFile)
testRaw <- read.csv(testFile)
dim(trainRaw)
dim(testRaw)
rm(trainFile)
rm(testFile)
```
The amount of observations between training and test sets are very different, but contains the same number of variables (160). In our case, the "classe" variable is the one to predict.


####Cleaning Data

_Clean the Near Zero Variance Variables._
```{r}
NZV <- nearZeroVar(trainRaw, saveMetrics = TRUE)
head(NZV, 20)
training01 <- trainRaw[, !NZV$nzv]
testing01 <- testRaw[, !NZV$nzv]
dim(training01)
dim(testing01)
rm(trainRaw)
rm(testRaw)
rm(NZV)
```

_Also, removing some columns of the dataset that do not contribute much to the accelerometer measurements._
```{r}
regex <- grepl("^X|timestamp|user_name", names(training01))
training <- training01[, !regex]
testing <- testing01[, !regex]
rm(regex)
rm(training01)
rm(testing01)
dim(training)
dim(testing)
```

_removing columns that contain NA's._
```{r}
cond <- (colSums(is.na(training)) == 0)
training <- training[, cond]
testing <- testing[, cond]
rm(cond)
```
Now, the cleaned training data set contains:
```{r}
dim(training)
#observations/variables
```

```{r}
dim(testing)
#observations/variables
```

##Correlation Matrix of Columns in the Training Data set.
```{r}
corrplot(cor(training[, -length(names(training))]), method = "color", tl.cex = 0.5)
```

###Partitioning Training Set
Was achieved by splitting the training data into a training set (70%) and a validation set (30%) using the following:
```{r}
set.seed(56789) # For reproducibile purpose
inTrain <- createDataPartition(training$classe, p = 0.70, list = FALSE)
validation <- training[-inTrain, ]
training <- training[inTrain, ]
rm(inTrain)
```

The training data set consist of
```{r}
dim(training)
```

The validation data set
```{r}
dim(validation)
```

and the Testing Data of
```{r}
dim(testing)
```

####Data Modeling
###Decision Tree
_Predictive model for activity recognition_
```{r}
modelTree <- rpart(classe ~ ., data = training, method = "class")
prp(modelTree)
```

_Performance of the model on the validation data set_
```{r}
predictTree <- predict(modelTree, validation, type = "class")
confusionMatrix(validation$classe, predictTree)
accuracy <- postResample(predictTree, validation$classe)
ose <- 1 - as.numeric(confusionMatrix(validation$classe, predictTree)$overall[1])
rm(predictTree)
rm(modelTree)
```

###Random Forest
_Being the training size 70 % of total dataset, the bias can not be ignored and k=5 is reasonable [Choice of K in K-fold cross-validation]https://stats.stackexchange.com/questions/27730/choice-of-k-in-k-fold-cross-validation)  _
```{r}
modelRF <- train(classe ~ ., data = training, method = "rf", trControl = trainControl(method = "cv", 5), ntree = 250)
modelRF
```
_Now, we estimate the performance of the model on the validation data set._

```{r}
predictRF <- predict(modelRF, validation)
confusionMatrix(validation$classe, predictRF)
accuracy <- postResample(predictRF, validation$classe)
ose <- 1 - as.numeric(confusionMatrix(validation$classe, predictRF)$overall[1])
rm(predictRF)
```

##Evaluate the model (out-of-Sample Error)
*Now, we apply the Random Forest model to the original testing data set downloaded from the data source. We remove the problem_id column first.*
```{r}
rm(accuracy)
rm(ose)
predict(modelRF, testing[, -length(names(testing))])
```

# Write the results to a text file for submission
```{r}
pml_write_files = function(x){
    n = length(x)
    for(i in 1:n){
        filename = paste0("problem_id_",i,".txt")
        write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
    }
}
```