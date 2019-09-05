---
layout: post
title: Exploring Azure AutoML with Genomics Data
#subtitle: Exploring AutoML
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

For an upcoming workshop I was looking for an interesting mix of genomics and machine learning to show off Azure capabilities. My [MS Genomics](https://azure.microsoft.com/en-in/services/genomics/) colleague [Erdal Cosgun](https://www.linkedin.com/in/erdal-cosgun-b75392134/) recently showed me his workshop material, which uses MS Genomics to predict variants and [Azure Machine Learning Studio](https://studio.azureml.net/) to predict (fake) phenotypes for the samples (in a fancy no-code GUI). I liked the idea and also wanted to try out something new: [Azure AutoML](https://docs.microsoft.com/en-us/azure/machine-learning/service/concept-automated-ml), a framework that originated in [Microsoft Research](https://www.microsoft.com/en-us/research/).

AutoML basically takes care of the  time-consuming, iterative tasks of ML model development. It trains & tunes a model, using the target metric you specify and iterates through multiple ML algorithms. Each iteration produces a model with a training score and everything is logged as an "Experiment" in the Azure portal. You could say it's a recommender system for ML pipelines that allows me to employ a range of ML techniques and practices without having to be an expert. It further has [preprocessing](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-create-portal-experiments#preprocess) (e.g. data imputation, encoding, embedding) and [interpretation capabilities](https://docs.microsoft.com/en-us/azure/machine-learning/service/machine-learning-interpretability-explainability) that can explain a models behaviour. The latter shows me for example which features where the most important ones. That follows the transparency principle, one of the six [Microsoft AI principles](https://blogs.partner.microsoft.com/mpn/shared-responsibility-ai-2/) (ever asked other cloud providers regarding theirs?). 

Initially I had planned to just use the AutoML [Visual Interface](https://azure.microsoft.com/en-in/blog/simplifying-ai-with-automated-ml-no-code-web-interface/) in the Azure Portal, so that I (and later the workshop participants) would not have to write any code. Unfortunately, the [variant call format (vcf)](https://en.wikipedia.org/wiki/Variant_Call_Format) is not really suitable for this. It usually has thousands of columns and the visual interface tries to preview all columns/features, which of course doesn't work at that scale. So instead I ran the analysis on an Azure ML Notebook VM using the AutoML Python API. 

I also decided to create my own input data: faked variant calls from a number of individuals and a limited number of sites, so that I could complete any analysis just on the Notebook VM. I made multiple sites "causal" and furthermore introduced a gender bias. The input data creation is scripted in the notebook itself and can be easily rerun to create new data with different features. The notebook shows how, with a few lines of Python, the AutoML system produces a highly performant model that picked the correct causal sites and gender as decisive factors for predicting a phenotype. The input data is obviously not fully representative of real data, but the capabilities of the AutoML platform are nicely demonstrated. This was fun and I hope the workshop participants will like this exercise.


The notebook covers:
- Input data generation
- Running AutoML
- Predicting outcome and plotting a confusion matrix
- Interpretation of the model

You can find the notebook here as [HTML rendered Notebook](/data/vcf-classification-04092019.html)
or on Github as [Jupyter Notebook](https://github.com/andreas-wilm/automl-with-genomicsdata/blob/master/automl-on-variants.ipynb). Unfortunately neither of these shows the AutoML widgets imported from the Azure Portal, so I added screenshots below:

#### AutoML Run Details

![AutoML Run Details Screenshot](/img/2019-09-04-automl-rundetails.png)

#### Explanation Dashboard

![CycleCloud Lustre GUI](/img/2019-09-04-automl-explanation-dashboard.png)



