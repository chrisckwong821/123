---
layout: post
header:
title: Numerai Tutorial - II - Label Specific Preprocessing and Iterative Screening
categories: [Machine Learning]
tags: [Numerai, Python, Data Science, Machine Learning]
fullview: true
comments: true
---

In the previous article, I have demonstrated the method to iteratively read in the data for Numerai tournament, implement data preprocessing, and  high-level algorithms from scikit learn by creating a class variable. I have also included an implementation of adversarial validation, which is to intentionally select training data which most resemble the test data. The assumption is that train and test sets may come from different distributions, and we are given a big set of training data relative to the test data that we can possibly waste some without losing too much information.

Machine learning is an exercise of garbage in and Garbage out("GIGO"). If you feed too complex data into a algorithm which does a lot of logics and maths, chances are you would get some meaningless output, as algorithms are at the end of days, merely a qunatiative representation of information. 

Therefore, I am going to dig into the preprocessing part in this article. For the tournament, the training and test sets come with a label call "era", which would determine our consistency score. If our prediction is consistent across all era, it would have a high consistency score, and vice versa. **Valid submission should have at least 75% consistency.**

So the logic is we can take advantage of this label to do some customizing for prediction, as long as we preserve a 75% consistency. In the last article, I have included a function that preprocesses the data era by era, however that funciton does not fit for the test data so I updated it with the following one:




```python
def eraStandardize(self,dataset,model):
        placeholder = pd.DataFrame()
        era = set([''.join(x for x in element if x.isdigit()) for element in dataset['era']])
        era.discard('')
        era = [int(i) for i in era]
        maxera,minera = max(era)+1,min(era)
        for i in range(minera,maxera):
            data = dataset[dataset['era']=='era{}'.format(i)][[f for f in list(dataset) if "feature" in f]]
            placeholder = placeholder.append(pd.DataFrame(model(data)))
        return placeholder
```


Since we need to feed in both training and test data for the era-specific preprocessing, this function would work with both data for convenience. we can specify the datasets by the variable `dataset`, and the function to be applied on the data by `model`.



```python
self.X_train = Preprocess().eraStandardize(self.training_data, Preprocess().StandardScaler)
self.X_test = Preprocess().eraStandardize(self.test_data, Preprocess().StandardScaler)
self.X_prediction = Preprocess().StandardScaler(self.X_prediction)
```


This is an example implementation. Both train and test data are fed into the custom eraStandardize function, using the scikitlearn `StandardScalar`, while the prediction data is fed into the StandardScalar function directly, since we dont have the `era` label for the prediction data.


Since the data is capable of adversarial validation, why don't we implement era-specific adversarial validation? So we screen out data for training era by era, in order to preserve the same percentage of data from each era.



```python
def advisory_screen(self,portion,train_x):
    
    model = RandomForestClassifier(n_estimators=50)
    
    X_test = self.x_prediction
    sample_size_test = X_test.shape[0]
    idholder = pd.DataFrame()
    
    for i in range(1,97):
        X_train = train_x[train_x['era']=='era{}'.format(i)][[f for f in list(train_x) if "feature" in f]]
        X_train_id = train_x[train_x['era']=='era{}'.format(i)].id.reset_index()
        sample_size_train = X_train.shape[0]
        X_data = pd.concat([X_train, X_test])
        Y_data = np.array(sample_size_train*[0] + sample_size_test*[1])
        
        model.fit(X_data,Y_data)
        pre_train = pd.DataFrame(data={'wrong-score':model.predict_proba(X_train)[:,1]})
        pre_test = pd.DataFrame(data={'right-score':model.predict_proba(X_test)[:,1]})
        num_data = round(portion * X_train.shape[0])
        test_alike_data = pd.concat([X_train_id,pre_train],axis=1)
        test_alike_data = test_alike_data.sort_values(by='wrong-score',ascending=False)[:num_data]
    ##############for control only#####
        print('out of {0} training sample and {1} testing sample'.format(sample_size_train,sample_size_test))
        print('correct for training: {}'.format(sum([1 for i in model.predict_proba(X_train)[:,1] if i<0.5])))
        #print('correct for validation: {}'.format(sum([1 for i in model.predict_proba(X_test)[:,1] if i>0.5])))
        #print(pd.concat([test_alike_data.head(n=5),test_alike_data.tail(n=5)]))
        #print(pd.concat([test_class.head(n=5),test_class.tail(n=5)]))
    #################################
        idholder = idholder.append(pd.DataFrame(test_alike_data.id),ignore_index=True)
    return train_x[train_x.id.isin(idholder.id)]

```

The code basically specifies the test data with a label of 1, and specify the training data with a label of 0. Then we apply the classifier to train the data era by era, select only the certain top range of training data. RandomForest with n_estimator 30 above performs quite well in the classification, usually misclassify only a few datapoints, but takes almost half an hour to complete one loop(all era for once) in my Mac when I use one core only.

If you want an iterative way to screen out data, while each time only screen out a small portion, you can easily do it by recursively feeding the data back into the function, until the number of data meets your target. In fact that is what I would recommend because it includes more stability and avoids screening out a lot of information in one go. However it would take quite a few hours to complete then.


```python

self.X = self.training_data
    while self.X.shape[0] >= 75000:
        self.X = Preprocess().advisory_screen(0.9,self.X)
        print(self.X.shape[0])
        
```

Let me know if you have any question/comment on the above code. I am also looking for teammates for Kaggle/Numerai or in general buddies to learn machine learning together. Contact me if you are interested in coding and machine learning. 

I am also open to do financial trading analysis on interesting topics. If you have a trade idea but having difficulty to implement the back-test/finding the relevant datasets, feel free to contact me and we can chat about it.
