---
layout: post
title: "Kaggle Competition Ensemble Model"
author: mike_chaykowsky
modified:
excerpt: "Final Ensemble Model"
tags: []
---
After the data imputation of the *Give Me Some Credit* Kaggle competition, I utilized the `caret` package to do some model tuning and determine optimal hyperparameters. Shown below are the XGBoost and GBM models and how I determined the hyperparameters for each.

## XGBoost tuning

XGBoost is also a tree-based method similar to Random Forest, however, xgboost utilizes boosting whereas random forest uses bagging. Both methods attempt to acheive lower error, but in different ways. Error comes from bias an variance. A very complex model has low bias and high variance, and a very simple model has high bias and low variance. Random forest attempts to reduce the variance of a very complex model of descision trees, which are each there own individual models. XGBoost reduces bias of a large number of weak learners where each of these weak learners will build upon each other in an almost nested fashion. XGBoost takes many parameters. Some of the parameters here were selected just to increase iterations towards the idea that if it runs longer it will perform better, and others has a more rigorous approach. First, all of the variables must have no factors, so data.matrix does this for us (also there really aren't any). We want to use a gradient boosting tree with a small step size, which eta is defined to be 0.01. max\_depth refers to the trees discussed earlier, this was fairly abitrarily set to 50 after observing the default value. Setting subsample to 0.5 means that XGBoost randomly collected half of the data instances to grow trees and this will prevent overfitting. This is essential to the model because this is the issue that random forest runs into. So XGBoost is a perfect balance of lack of overfit by applying these weak learners on random resamples of the data. Then since we set subsample to 0.5, we also have to set colsample\_bytree to 0.5. These parameters are passed into xgboost and then the number of iterations was set high to 500 and verbose = 1 just means that we get to see how the model progresses as it's running based on the training AUC value.


{% highlight r %}
xgbModel <- function(dfTrain, dfTest) {
  test_names <- names(dfTest)
  train_names <- names(dfTrain)
  intersect_names <- intersect(test_names, train_names)
  
  training <- xgb.DMatrix(data = data.matrix(dfTrain[intersect_names]),label=(dfTrain[,c("SeriousDlqin2yrs")]))
  
  # subsample - setting it to 0.5 means that XGBoost randomly collected half of the data instances to grow 
  # trees and this will prevent overfitting. 
  # colsample_bytree - subsample ratio of columns when constructing each tree.
  param <- list(objective = "binary:logistic", booster = "gbtree", eta = 0.01, max_depth = 50,
                subsample = 0.5, colsample_bytree = 0.5)
  
  train.xgb <- xgboost(data = training, params = param, nrounds = 500, verbose = 1, eval_metric="auc")
  
  train.xgb
}
{% endhighlight %}

## GBM tuning

The `caret` package allows you to cross-validate on multiple hyperparameters and define a metric to check for the optimal values. Since Kaggle uses AUC (or area under the ROC curve), I utilized the ROC metric for tuning. I asked the function to check for interaction.depth and shrinkage. Any more than that and this code will run for a very long time. You can see the output displayed below. The labels were given names "good" and "bad" so the results are legible.


{% highlight r %}
require(caret)
Train$SeriousDlqin2yrs <- as.factor(Train$SeriousDlqin2yrs)
levels(Train$SeriousDlqin2yrs) <- c("Good", "Bad")
caretGrid <- expand.grid(interaction.depth=c(5, 10), n.trees = 500,
                   shrinkage=c(0.01, 0.001), n.minobsinnode = 5)
metric <- "ROC"
trainControl <- trainControl(method="cv", number=10, classProbs = TRUE)
gbm.caret <- train(SeriousDlqin2yrs ~ ., data=Train, distribution="bernoulli", method="gbm",
              trControl=trainControl, verbose=TRUE, 
              tuneGrid=caretGrid, metric=metric, bag.fraction=0.3)

# Stochastic Gradient Boosting 
# 
# 150000 samples
#     10 predictor
#      2 classes: 'Good', 'Bad' 
# 
# No pre-processing
# Resampling: Cross-Validated (10 fold) 
# Summary of sample sizes: 135000, 135000, 134999, 134999, 135001, 135000, ... 
# Resampling results across tuning parameters:
# 
#   shrinkage  interaction.depth  Accuracy   Kappa    
#   0.001       5                 0.9331600  0.0000000
#   0.001      10                 0.9331600  0.0000000
#   0.010       5                 0.9374467  0.2608611
#   0.010      10                 0.9376333  0.2711224
# 
# Tuning parameter 'n.trees' was held constant at a value of 500
# Tuning parameter 'n.minobsinnode' was
#  held constant at a value of 5
# Accuracy was used to select the optimal model using  the largest value.
# The final values used for the model were n.trees = 500, interaction.depth = 10, shrinkage = 0.01
#  and n.minobsinnode = 5.
{% endhighlight %}

Here I loaded in the dataframes from the data imputation post.


{% highlight r %}
library(dplyr)
library(e1071)
library(randomForest)
library(gbm)
library(xgboost)
Train <- read.csv("training_imputed.csv")
Test <- read.csv("test_imputed.csv")
Train$X <- NULL
Test$X <- NULL
Test$SeriousDlqin2yrs <- NULL
{% endhighlight %}


{% highlight r %}
## Gradient Boosting Machine model as defined above in model tuning section
#    args: dfTrain -> the training dataframe required
# --------------------------------------------------------------
GBMModel <- function(dfTrain) {
  GB <- gbm(dfTrain$SeriousDlqin2yrs ~ ., data=dfTrain, n.trees=5000,
            keep.data=FALSE, shrinkage=0.01, bag.fraction=0.3,
            interaction.depth=10)
  GB
}
{% endhighlight %}


{% highlight r %}
## Random Forest model
#    args: dfTrain -> the training dataframe required
# --------------------------------------------------------------
forestModel <- function(dfTrain) {
  train.Forest <- randomForest(formula = as.factor(SeriousDlqin2yrs) ~ ., 
                               data = dfTrain, ntree = 2000, importance = TRUE, 
                               na.action=na.omit)
  train.Forest
}
{% endhighlight %}

There's a lot behind the choices of kernel in a support vector machine, so I think I will save this discussion for a separate post and focus on the model development here. I also ended up not even using this model in the ensemble so it is only here to show that SVM does not work well for this question.


{% highlight r %}
## Support Vector Machine model
#    args: dfTrain -> the training dataframe required
# --------------------------------------------------------------
svmModel <- function(dfTrain) {
  train.svm <- svm(as.factor(SeriousDlqin2yrs) ~ ., 
                   type='C-classification', kernel='radial', 
                   cachesize=2000, probability=TRUE, cost=1, 
                   data = dfTrain)
  train.svm
}
{% endhighlight %}

The XGBoost was discussed above along with the hyperparameters.

{% highlight r %}
## Support Vector Machine model
#    args: dfTrain -> the training dataframe required
#          dfTest -> the test dataframe required
# --------------------------------------------------------------
xgbModel <- function(dfTrain, dfTest) {
  test_names <- names(dfTest)
  train_names <- names(dfTrain)
  intersect_names <- intersect(test_names, train_names)
  
  training <- xgb.DMatrix(data = data.matrix(dfTrain[intersect_names]),label=(dfTrain[,c("SeriousDlqin2yrs")]))
  
  # subsample - setting it to 0.5 means that XGBoost randomly collected half of the data instances to grow 
  # trees and this will prevent overfitting.
  # colsample_bytree - subsample ratio of columns when constructing each tree.
  param <- list(objective = "binary:logistic", booster = "gbtree", eta = 0.01, max_depth = 50,
                subsample = 0.5, colsample_bytree = 0.5)
  
  train.xgb <- xgboost(data = training, params = param, nrounds = 500, verbose = 1, eval_metric="auc")
  
  train.xgb
}
{% endhighlight %}

Here I put together an ensemble of all the previous models described above. The SVM is commented out because it did not improve AUC score when submitted to Kaggle.


{% highlight r %}
## Ensemble model
#    args: dfTrain -> the training dataframe required
#          dfTest -> the test dataframe required
# --------------------------------------------------------------
ensemble <- function(dfTrain, dfTest) {
  #SVM commented out because does not improve AUC score
  models <- list(rfs=forestModel(dfTrain),
                           gbs=GBMModel(dfTrain),
                           #svm=svmModel(dfTrain),
                           xgb=xgbModel(dfTrain, dfTest))
  models
}
{% endhighlight %}

The `ensemblePredictions()` function will output the final prediction probabilities to a `.csv` file ready to be submitted to Kaggle. The main aspect here that needs improvement is the weights on the individual models. I simply upweighted the most effective models based on their individual AUC score in the ensemble of all three. The XGBoost performed the best so it was upweighted the most, then the GBM, then the RF. These weights are fairly arbitrary and more trial and error than anything else.


{% highlight r %}
## Support Vector Machine model
#    args: model -> the ensemble model used to predict
#          dfTrain -> the training dataframe required
#          dfTest -> the test dataframe required
# --------------------------------------------------------------
ensemblePredictions <- function(model, train, test) {
  pred <- predict(model$rfs, test, type = 'prob')[,2]
  pred2 <- predict(model$gbs, test, n.trees = 5000)
  #pred3 <- predict(model$svm, test)
  
  test_names <- names(test)
  train_names <- names(train)
  intersect_names <- intersect(test_names, train_names)

  testDataXGB <- xgb.DMatrix(data = data.matrix(test[,intersect_names]))
  pred4 <- predict(xgb1,testDataXGB)
  
  pred <- data.frame(Id = seq.int(nrow(test)), ProbabilityRF=pred, 
                     ProbabilityGBM=pred2, ProbabilityXGB=pred4)
  pred$ProbabilityGBM <- 1/(1+exp(-pred$ProbabilityGBM)) # sigmoid function to convert to probability
  
  # After submitting each model individually, this upweights models with greater individual AUC
  pred$Probability <- ((pred$ProbabilityGBM)*4 + pred$ProbabilityRF + (pred$ProbabilityXGB)*20)/25
  
  pred <- pred %>% dplyr::select(Id, Probability)
  write.csv(pred, file="probabilities4-1-20train.csv", row.names=FALSE)
}
{% endhighlight %}


{% highlight r %}
ensemble.models <- ensemble(Train, Test)
ensemblePredictions(ensemble.models, Train, Test)
{% endhighlight %}

![center](/images/2017-03-09-Credit-Risk-Kaggle/ranking150.png)
