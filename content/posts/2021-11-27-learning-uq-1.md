---
title: "Uncertainty quantisation: Chapter 1"
date: 2021-11-17T16:02:29Z
draft: true
---

I have been watching some of the excellent recordings from Pydata Global 2021, and one of the tutorials was about model uncertainty quantisation, with demos using the [IBM UQ360 library](https://uq360.mybluemix.net/). This is something I have been meaning to learn how to implement properly for a while, so found it very useful! In this series of posts, I will try to summarise the main learnings from that tutorial and other readings I did around uncertainty quantification techniques.

## What is uncertainty quantification?
When we build machine learning models, there is always a degree of error in the predictions -- there two main types:
- **model uncertainty** (also known as epistemic uncertainty):  this can be reduced by more data-- for example, if the training data only covers the months june-decemeber, and you need to predict effects in jan-may, you can reduce the uncertainty in your predictions by gathering more data from jan-may
- **data uncertainty** (also known as aleatoric uncertainty): this cannot be reduced by adding more data, and is caused by inherent randomness, e.g. there is always uncertainty when you are throwing a dice)

Uncertainty quantification the process of measuring this error and to communicate this clearly to the end user. There are also methods to measure how good your uncertainty estimates are, and adjust as necessary.

### Methods of estimating Uncertainty
1. 'Intrinisic' methods-- the uncertainty estimation is built into the model training process. For example, the crossentropy loss we use in neural nets give an estimate of data uncertainty. For regression tasks, there's methods such as Quantile Regression which also gives an estimate of data uncertainty. More advanced methods such as Bayesian networks can also capture model uncertainty. The methods offered by UQ360 to train models with intrisinic estimates are:
    - Bayesian neural nets (regression and classification)
    - Quantile regression
    - Homoscedastic Gaussian Process Regression
    - Heteroscedastic Regression
    - Ensemble Heteroscedastic Regression
    - Actively Learned Model
2. 'Extrinisic' methods-- these are applied to a trained model that do not offer uncertainty estimates in a blackbox like fashion
    - Auxiliary Interval Predictor
    - Blackbox Metamodel Classification
    - Blackbox Metamodel Regression
    - Infinitesimal Jackknife
    - Classification Calibration
    - UCC Recalibration
    - Structured Data Predictor
    - Short Text Predictor
    - Confidence Predictor

(I will try to explain my understanding of these algorithms are in following posts)

## Exploring the librarie's functionality
It's a lot easier to see how something work with examples, and while there are official tutorials and demos provided [here](https://github.com/IBM/UQ360/tree/main/examples), I would like to get it to work on my own dataset to see where the pain points are. For the first example on regression, I will be using an Airbnb dataset for London, 