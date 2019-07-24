---
title: "Machine Learning Final"
author: "Erik Eastman"
date: "June 21, 2019"
output:
  pdf_document: default
  html_document: default
---


##Machine Learning - What is proper exercise?


```r
library(dplyr)
library(caret)
library(RColorBrewer)
library(rmarkdown)
library(ranger)
library(ggplot2)
```


I first read in the data set and looked at the variables. There were some that were sparse in the training set and did not have any values in the test set. These were prime for elimination from the model as there are 160 total columns in the dataset so nearly that many predictors. I used the dplyr package to start cutting down the dataset.


```r
trainingSetDL <- read.csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv")
##Removed these fields. Only available in about 400 observations out of 19K. Reduces possible predictors from 159 to 59.
##No real difference in proportions of counts of type "Classe" in those with "Window_New" = Yes to overall population.
trainingSetDL <- trainingSetDL %>% select(-starts_with("kurtosis")) %>%
                                   select(-starts_with("skewness")) %>% 
                                   select(-starts_with("max")) %>%
                                   select(-starts_with("min")) %>%
                                   select(-starts_with("amplitude")) %>%
                                   select(-starts_with("var")) %>%
                                   select(-starts_with("avg")) %>%
                                   select(-starts_with("stddev")) %>%
                                   select(-contains("timestamp")) %>%
                                   select(-"user_name") %>%
                                   select(-"new_window") %>%
                                   select(-"X")
```


I printed out a correlation matrix to quickly manipulate and view in Excel to see if anything jumped out at me. Some items were highly correlated. I wanted to run several models with all variables first before I started to remove any more variables but I had a plan if I needed it.


```r
corMatrix <- round(cor(trainingSetDL[,2:53]),2)
writexl::write_xlsx(as.data.frame(corMatrix), path = "~/corMatrix.xlsx")
```

I created a training set and validation set to estimate how my various models to test may do on the actual training set (since we don't know the 'classe' outcome). I landed on a 70/30 split since we have a pretty large dataset.


```r
set.seed(5678)

## Create Set to build model
inBuild <- createDataPartition(y=trainingSetDL$classe, p = .7, list = FALSE)

buildSet <- trainingSetDL[inBuild,] 

##Create Set to validate modesl
validationSet <- trainingSetDL[-inBuild,]
```



I Read in the testing set of 20 observations to predict against.


```r
testingSetDL <- read.csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv")

##Don't know outcome of test set but will need to append predictions to this set for peer scoring
testingSetDL <- testingSetDL %>% select(-starts_with("kurtosis")) %>%
  select(-starts_with("skewness")) %>% 
  select(-starts_with("max")) %>%
  select(-starts_with("min")) %>%
  select(-starts_with("amplitude")) %>%
  select(-starts_with("var")) %>%
  select(-starts_with("avg")) %>%
  select(-starts_with("stddev")) %>%
  select(-contains("timestamp")) %>%
  select(-"user_name") %>%
  select(-"new_window")
```


I first tried and Linear Descriminant Analysis model since I figured it would run faster than a Random Forest or Boosting model. I used all defaults first. I tested the "Build Set/Training Set" against the "Validation Set". I printed out the Variable Importance data (not shown) and the confusion matrix (shown). The accuracy of this model was 72%. The accuracy is not high enough to pass the 80% threshold required by the quiz. I needed a different model. No amount of tuning was probably going to get me to 80%. (Although some other feature selection could get me there.)


```r
modelLM <- train(classe ~ .,
                 data = buildSet,
                 method = "lda")

importanceLM <-  varImp(modelLM)

predictionLM <-  predict(modelLM, validationSet)


confusionLM <- confusionMatrix(predictionLM, validationSet$classe) 
confusionLMColor <- RColorBrewer::brewer.pal(5,"RdYlGn")

plot(confusionLM$table, xlab = "Reference", ylab = "Prediction", main = "LDA Model Confusion Matrix", col = confusionLMColor)
```

![Confusion Matrix](figure/Linear Descriminant Analysis Model-1.png)

I next decided to try a Random Forest model. In Caret the "ranger" model does parallel processing much like XGBoost without having to do hot encoding.



```r
# modelRF <- train(classe ~ ., 
#                  data = buildSet, 
#                  method = "ranger")

# predictionRF <- predict(modelRF, validationSet)
confusionRF <- confusionMatrix(predictionRF, validationSet$classe) ##Accuracy of 99%
confusionRF
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction    A    B    C    D    E
##          A 1673    1    0    0    0
##          B    1 1137    1    0    2
##          C    0    1 1025    3    0
##          D    0    0    0  960    0
##          E    0    0    0    1 1080
## 
## Overall Statistics
##                                           
##                Accuracy : 0.9983          
##                  95% CI : (0.9969, 0.9992)
##     No Information Rate : 0.2845          
##     P-Value [Acc > NIR] : < 2.2e-16       
##                                           
##                   Kappa : 0.9979          
##                                           
##  Mcnemar's Test P-Value : NA              
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            0.9994   0.9982   0.9990   0.9959   0.9982
## Specificity            0.9998   0.9992   0.9992   1.0000   0.9998
## Pos Pred Value         0.9994   0.9965   0.9961   1.0000   0.9991
## Neg Pred Value         0.9998   0.9996   0.9998   0.9992   0.9996
## Prevalence             0.2845   0.1935   0.1743   0.1638   0.1839
## Detection Rate         0.2843   0.1932   0.1742   0.1631   0.1835
## Detection Prevalence   0.2845   0.1939   0.1749   0.1631   0.1837
## Balanced Accuracy      0.9996   0.9987   0.9991   0.9979   0.9990
```

```r
modelRF$finalModel$prediction.error ##Out of Bag Error Rate = .00175 
```

```
## [1] 0.001747106
```

```r
confusionRFColor <- RColorBrewer::brewer.pal(5,"RdYlGn")
qplot(confusionRF$table, xlab = "Reference", ylab = "Prediction", main = "Random Forest Model Confusion Matrix", col = confusionRFColor)
```

```
## Error: Aesthetics must be either length 1 or the same as the data (25): colour
```

![plot of chunk Random Forest Model](figure/Random Forest Model-1.png)

```r
finalPredictionRF <- predict(modelRF, testingSetDL)
##Out of Sample error will most likely be 0% or could be 5% in a small fraction of cases.
finalPredictionRF ## B, A, B, A, A, E, D, B, A, A, B, C, B, A, E, E, A, B, B, B = 20/20 on quiz
```

```
##  [1] B A B A A E D B A A B C B A E E A B B B
## Levels: A B C D E
```

```r
##True out of sample error in this case was 0%!
```



