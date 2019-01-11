---
title: "PU Learning"
author: "Philip Massie"
date: 2019-01-11
draft: false
tags: [ "PU learning", "machine learning", "data science" ]
output: 
  html_document: 
    keep_md: yes
    toc: no
---

## Introduction
Disclaimer. not intended to be exhaustive and complete. Really just my notes but may be helpful to others. this post is based on this Fusilier et al. 2014 /cite
Outline the problem in general.

previously I have used the approach, in a highly imbalanced dagta set, of randomly sampling from the unkowns and treating these as negatives. I freely admit that this is not exactly ideal, but out in the wild, with real-life deadlines, the approach "wasn't the worst". This challenge kwwps cropping up int the data sets I typicqlly work with and in the questions I am often persuing. So I was recently ble to take a few moments to read around the topic a little and it seemed like it might be worth keeping a few notes:

There are a few different PU approaches around.
Most widely cited initial attempts come from Liu et al. 2002 and 2003 but I’m not absolutely sure if they were the first to try and isolate an approach to the problem.
Interesting diffs between the two are that while Liu et al/'s approach iteratively increases the size of the negative class from the unknowns, Fusilier et al.s approach iteratively reduces and refines it. That means that it may prove useful for highly imbalanced class problems and the authors do point this out.

Other PU approaches involving imbalanced data often involve some sort of random sampling or bagging approach, but Fusilier et al.'s approach rely on the classifier itself to isolate so called reliable negatives (RN) from the unknown data set.

One such sapling approach to PU learnign involves bagging. Basically, random bootstrap samples are drawn from the unknown set, and classifiers are trained to distinguish between the positive samples and the various bootstrap unknown sets. each random set should have dfferent degrees of contamination by unlabled positives and .. **flesh this out a bit**

## Methods

### 'Original' approach (Liu et al. 2002 and 2003)
Given a training set containing only positives (P) and unknown (U) classes follow the following steps:

1. Treating all U as negatives (N) train a classifier P vs. U
2. Using the classifier, score the unknown class and isolate the set of 'reliable' negatives (RN).
3. Train a new classifier on P vs. RN, use it to score the remaining U and isolate additional RN.
4. Repeat step 3, iteratively enlarging the set of RN until the stopping condition is met. 

The stopping condition is met when no new negative cases are classified.

Where `Q` is defined as the set of unknowns classified as negatives and `i` is the iterator, the stopping condition is defined as:

```|Qi| > 0```

### Modified approach (Fusilier et al. 2014)
Given a training set containing only positives (P) and unknown (U) classes follow the following steps:

1. Treating all U as negatives (N) train a classifier P vs. U
2. Using the classifier, score the unknown class and isolate the set of 'reliable' negatives (RN).
3. Train a new classifier on P vs. RN. Score RN and exclude predicted positives from RN
4. Repeat step 3, iteratively refining the RN set, until the stopping condition is met. 

The stopping condition ensures that Q reduces in size (avoiding sudden large reductions in RN size) while the RN set never gets smaller than the P set. More explicitly:

>while the size of the set of unknowns classified as negatives in _this iteration_ is smaller than or equal to the size of the set of unknowns classified as negatives in _the previous iteration_ and the size of the set of positive classes  is smaller than the set of refined RNs resulting from _this iteration_

Where `Q` is defined as the set of unknowns classified as negatives and `i` is the iterator, the stopping condition is defined as:

```|Qi| <= |Q(i-1)| & |P| < |RNi|```

> #### Things to follow up on:
> 1. Most of the articles use SVM, but they also tend to be NLP problems. Does the classifier family matter much?
> 2. How does the original paper identify the cut-off for determining 'reliability'
> 3. Modified approach: Why could Q get larger with the iterations?
> 4. Bagging approach: Consider how best to penalise false negatives.
>       - Cutoff selection?

### Bagging approach (Inductive) (Mordelet & Vert 2013)
Given a training set containing only positives (P) and unknowns (U), where K = size of bootstrap samples and T = number of samples, follow the following steps:

1. Draw a bootstrap sample Ut of size K from U
2. Train a classifier P vs Ut
3. Repeat steps 1 and 2 T times
4. Score the test data with an ensemble approach using the bagged models.

The stopping criterion here is determined by the value of T and the authors suggest that there is typically not much additional value to be gained by setting T > 100. Judging from their plots however, where `|P|` and `K` are both large large, there is little change above T = 5. I suspect it's worth trying to keep track of this during training if possible or setting up an early stopping type criterion in your function because depending on your time constraints, training 100 models may not be viable.

## Conclusion
These three methods provide sensible approaches to the problem of PU learning but only the modified and bagging approaches provide inherent ways to deal with imbalanced data. My plan is to try and implement these 2 approaches and compare their results. While I cant share the data publicly, I will try and share the code and general results on the blog and in GitHub. We work primarily in Python/PySpark or Scala/Spark. Since we don't have friendly interactive environments at work for Scala I tend to do EDA and testing in PySpark before moving everything over to Scala. If you have nice Scala/Spark environments set up check out 

- https://github.com/ispras/pu4spark PU learning libraries written in Spark
- https://astrakhantsev.com/pu-learning/ nice post written by author of pu4spark
- https://roywright.me/2017/11/16/positive-unlabeled-learning/ Nice overview of PU learning approaches

## References
Fusilier DH, Montes-y-Gómez M, Rosso P, Guzmán Cabrera R (2015) Detecting positive and negative deceptive opinions using PU-learning. Inf Process Manag 51:433–443. doi: 10.1016/j.ipm.2014.11.001

Liu B, Dai Y, Li X, et al (2003) Building text classifiers using positive and unlabeled examples. In: Third IEEE International Conference on Data Mining. pp 179–186

Liu B, Lee WS, Yu PS, Li X (2002) Partially Supervised Classification of Text Documents. In: Proc. 19th Intl. Conf. on Machine Learning. pp 387–394

Mordelet F, Vert J-P (2014) A bagging SVM to learn from positive and unlabeled examples. Pattern Recognit Lett 37:201–209. doi: 10.1016/j.patrec.2013.06.010



