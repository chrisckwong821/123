---
layout: post
header:
title: Numerai Tutorial - I - Vanilla Algorithms and Adversarial Validation
subtitle: plug-and-play machine learning
categories: [Machine Learning]
tags: [Numerai, Python, Algorithm]
header-img: "img/post-bg-universe.jpg"
fullview: true
comments: true
---

[Numerai](https://numer.ai/) Competition is an online machine learning tournament which is operated by Numerai, a hedge fund. You may refer to this [article](https://www.wired.com/2017/02/ai-hedge-fund-created-new-currency-make-wall-street-work-like-open-source/) for an introduction. In this article, I want to describe how I approach this problem, and built up a set of tools that helps me to rapidly iterate over different algorithms for testing, feature preprocessing and engineering.


```python
import os.path
import pandas as pd
class models:
    def __init__(self,tournament=70):
        parentdir = os.path.join(os.path.abspath(os.getcwd()),os.pardir)
        path = os.path.join(parentdir,"T{}/".format(tournament))
        
        self.training_data = pd.read_csv(path + "numerai_training_data.csv",header=0)
        self.test_data = pd.read_csv(path + "numerai_tournament_data.csv",header=0)
print(models().training_data.head(1))        
```
<script src="https://gist.github.com/chrisckwong821/0ea85216a9b9b158334e24ded809a881.js"></script>

The data contains 21 features range from 0 to 1, with the binary target(0,1). Numerai claims that they encrypted financial data into the dataset so it is more than simple time-series data. Each row contains a unique id, an era which label its type, and whether it belongs to train/test data.


```python
#training data
        self.X_train = self.training_data[[f for f in list(self.training_data) if "feature" in f]]
        self.y_train = self.training_data['target']
#test data(part of prediction data)
        self.X_test = self.test_data[[f for f in list(self.training_data) if "feature" in f]][:16686]
        self.y_test = self.test_data['target'][:16686]
#prediction data
        self.X_prediction = self.test_data[[f for f in list(self.training_data) if "feature" in f]]
```

I inititize the training,test and prediction data under the class init for reference. Noted that the test data is part of the prediction data here. This would measure the error in a way more relevant to the final output.
Still, cross validation would still be used during the training, simple because it would make the training more robust.


```python
    def DTC(self):
        model = DecisionTreeClassifier()
        p = [{'min_samples_split':[[2]],'max_features':[['log2'],['auto']],'max_depth':[[5]]}]
        return model,p
    def RFC(self):
        model = RandomForestClassifier()
        p = [{'n_estimators':[[10,50,100,300]],'min_samples_split':[[2]],'max_features':[['log2'],['auto']],'max_depth':[[2,3,4]]}]
        return model,p
    def LR(self):
        model = LogisticRegression()
        p = [{'max_iter':[[1000]],'tol':[[0.00001]]}]
        return model,p
```

To manage each model and its parameter efficiently, each model is wrapped under a function. In this format, basically I would be able to call any models from scikit-learn in this format efficiently and tune the parameters as I want. The model and its parameter would then be fed into a kernel function for training. 


```python
    def kernel(self,model,p):
            parameter = ParameterGrid(p)
            clf = GridSearchCV(model, parameter, cv=3, scoring='neg_log_loss',n_jobs=2)
            clf.fit(self.X_train,self.y_train)
            # proba is the prediction of the final prediction data
            prediction = clf.predict_proba(self.X_prediction)[:,1]
            # part of the data is used to calculate the logloss for measuring performance 
            error = log_loss(self.y_test,prediction[:16686],normalize=True)
            print(error)
            print(clf.best_params_)
            result = self.ids_test.join(pd.DataFrame(data={'probability':prediction}))
            result.to_csv('%.4f_submission.csv'%(error),index=False)
            #return result,error
```

Prediction is the model predicted probability range from 0 to 1. It is joined with the ids to form a standardized table readily for submission at Numerai. The performance would be measured by the logloss to the ground truth. This is usally close to your performance measured by the test score from my experience.

The above part forms the basic and skeleton part of the workflow. Obviously plugging vanilla algorithms into the dataset is not going to get your far. Depending on which model you feed, it would rarely get you farther than 0.6923. RandomForest and Logistic Regression are among the best. But still, only marginally better than 0.6931/-log(0.5) if you guess 0.5 for all input, contrast to the level above 0.6880 for people on top 100.

To incorporate feature preprocessing and engineering into the workflow, we can initilize some preprocessing functions:


```python
    def StandardScaler(self,x): #to unit variance
        model = preprocessing.StandardScaler(copy=False)
        return model.fit_transform(x)
    def PolyFeature(self,x):
        model = preprocessing.PolynomialFeatures(interaction_only=False)
        return pd.DataFrame(model.fit_transform(x))
    def KernelCenterer(self,x): #only demean
        model = preprocessing.KernelCenterer()
        return pd.DataFrame(model.fit_transform(x))
```

Similar to what is done for the model, define the API under a function and return the transformed dataframe, noted that the model.fit_transform returns an numpy array thus have to be wrapped as a DataFrame for later processing.

Inspired by [this article](http://fastml.com/adversarial-validation-part-one/) on adversarial validation, I have implemented this method as well. Basically it trains a classifier to tell training data from prediction data, then use the training data that most resemble the prediction data.


```python
    def advisory_screen(self,samplesize=10000):
        model = RandomForestClassifier(n_estimators=50)
        X_train = self.training_data.drop(['id','era','data_type','target'],1)
        X_test = self.X_prediction[16686:]
        model.fit(X_data,Y_data)
        
        pre_train = pd.DataFrame(data={'wrong-score':model.predict_proba(X_train)[:,1]})
        pre_test = pd.DataFrame(data={'right-score':model.predict_proba(X_test)[:,1]})
        test_alike_data = pd.DataFrame(self.ids_train).join(pre_train).sort_values(by='wrong-score',ascending=False)[:samplesize]
        test_class = self.ids_test[16686:].reset_index().join(pre_test).sort_values(by='right-score',ascending=False)
        Y_train = self.training_data[['id','target']]
        
        #####just for control
        print('out of {0} training sample and {1} testing sample'.format(sample_size_train,sample_size_test))
        print('correct for training: {}'.format(sum([1 for i in model.predict_proba(X_train)[:,1] if i<0.5])))
        print('correct for validation: {}'.format(sum([1 for i in model.predict_proba(X_test)[:,1] if i>0.5])))
        print(pd.concat([test_alike_data.head(n=5),test_alike_data.tail(n=5)]))
        print(pd.concat([test_class.head(n=5),test_class.tail(n=5)]))
        ########
        
        ids = test_alike_data.join(Y_train.set_index('id'),on='id')['id']
        return self.X[self.training_data['id'].isin(ids)],self.Y[self.training_data['id'].isin(ids)]

```

Basically the function outputs the X and Y that most resembles the prediction data, of custom range. I use default sample size of 10000 which is reported by others to be most efficient. But this is heuristic and can be tested case by case.

After this adviserial screening, the model actually does not improve significantly. My best score is 0.6920 only. To further take advantage of the data given, I want to do some averaging on the "era" label. For each training data, there are 96 era of varying sizes. which may range from one hundred to one thousand. To use this, I define a new function that 


```python
    def eraStandardize(self,model):
        placeholder = pd.DataFrame()
        for i in range(1,97):
            data = self.training_data[self.era=='era{}'.format(i)][[f for f in list(self.training_data) if "feature" in f]]
            placeholder = placeholder.append(pd.DataFrame(model(data)))
        return placeholder
```

Model is the function from which we modify the data era by era. We can apply demean, unit variance, minmaxscaler or binarizer into the model. 

I am still working on ways to improve the performance, incorporate codes that allow faster iteration and testing. I have not incorporated model ensembling, mainly because my score is far from good, and ensembling a bunch of bad models would not make a good one. So I would also release my ensembling code after I manage to make some improvement on the logloss.

Let me know anyway to improve and any feedback is welcome!
