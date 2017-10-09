---
layout: post
header-img: "img/amazing.jpg"
title: Pair Trading - II - Regression and Ornstein-Uhlenbeck process(O-U Model)
categories: [Machine Learning, Financial Trading]
tags: [Python, Machine Learning, Pair Trading]
fullview: true
comments: true
---
This is a continuation of the Pair Trading Series. For the first article which discusses how to find out potential trading pairs using clustering DBSCAN, you may go back to [here](https://chrisckwong821.github.io/machine%20learning/financial%20trading/2017/09/04/Pair-Trading-1.html). 

In this article, I would focus on the analysis of trading pairs by modelling a mean-reverting spread. There are a few ways to define "spread" between prices, I would list a fews that I have come across:

(1) Simple price differences between two products: 

This only gives a sense of how two products move relative to one another. Practically, unless two products have similar price scales, the simple difference has not much meaning.

(2) Percentage Change:

Normalize the simple price difference from a chosen point in time, then measure the spread as if a product itself. This has the advantage of comparison with previous data. The downside is, it is not representative of the pair relationship. For example, assume there are stock A valued at $1 and stock B valued at $10 at time t0. At time t1, both appreciated 10% so $1.1 and $11 for stock A and stock B respectively. However, the spread would appreciated 10% from $9 to $9.9 as well. So practically speaking, simple price difference/its standardized form both would not converge to a particular level. 

(3) Relative Percentage Change:

Using the previous example, an improved form would be measuring the percentage change of both products. So if both stock A and stock B move up 10%, their spread movement would be 0%. This is a more realistic approach to model spread. In pair trading, where we long one product and short the others, profit and loss would be dependent on this relative percentage change given the portfolio is constructed evenly among two stocks. For example, assume one million stock A is longed and one million stock B is short sold, stock A has a 10% gain while stock B has a 5% gain. Then the pnL would be 0.5x1.1+0.5x0.95 = +%2.5. So a spread of (5%) results in a +2.5% gain in the entire portfolio. From this example, the percentage spread is directly representative of gain from this type of simple long/short.

(4) Price Ratio:

Directly divide the price of one stock by the other one. This measures the geometric change of both prices. This is the most common use of spread.


In this article, I would implement a model that assumes a mean-reverting behavior hidden in the price differential defined by (3). I took reference from [the paper written by Y Chen, W Ren and X Lu](http://cs229.stanford.edu/proj2012/ChenRenLu-MachineLearningInPairsTradingStrategies.pdf) from Stanford. The paper illustrated two approaches to model spread, one adopts O-U model and regression and the other one implemented Kalman's Filter and Expectataion Maximization. For clarity, I would implement the first approach in this article and separate the other one in the next article. I would follow the paper step-by-step with comment on codes. At the end, a profit and loss path would be simulated to measure how  the stock pair is performing.


In the last article, the stock pair 0386.HK and 0857.HK was discovered by the clustering algorithm DBSCAN. Let's use this as an example for our analysis:



```python

import os
import pandas as pd
target = ['0386.HK','0857.HK']
path = os.path.join(os.getcwd()+'/'+os.path.pardir,"part1/StockData/")
df = pd.DataFrame()
openprice = pd.DataFrame()
window=60
for i in target:
    data = pd.read_csv(path+'{}.csv'.format(i))
    data.set_index('Date',inplace=True)
    a = data['Adj Close']
    b = data['Open']
    openprice = pd.concat([openprice,b.rename('{}'.format(i))],axis=1)
    df = pd.concat([df,a.rename('{}'.format(i))],axis=1)

openprice.fillna(method='ffill',inplace=True)
df.fillna(method='ffill',inplace=True)
print(df.head(2))
print(openprice.head(2))

```

                 0386.HK   0857.HK
    Date                          
    2015-09-04  4.421465  5.657228
    2015-09-07  4.366312  5.618280
                0386.HK  0857.HK
    Date                        
    2015-09-04     4.93     5.94
    2015-09-07     4.72     5.75



Let's get our dataset ready by reading in the stock data downloaded from Yahoo Finance. The first dataframe is closed price which would be used for signal generation, while the second is the opening price which would be used for profit and loss simulation. Recalled from last article that a crawler was writtern to get the information. You may download the data(As of 04-09-2017) from [here](https://github.com/chrisckwong821/chrisckwong821.github.io/blob/master/assets/Reference/StockData.zip).




```python

import statsmodels.api as sm
def rollingreg(df,window=window,target=target):
    alpha,beta = [],[]
    for i in range(df.shape[0]-window+1):
        X = sm.add_constant(list(df['{}'.format(target[0])][i:i+window]))  ### target[0] is X
        Y = list(df['{}'.format(target[1])][i:i+window])
        model = sm.OLS(Y,X)
        result = model.fit().params
        alpha.append(result[0])
        beta.append(result[1])
    alpha = pd.DataFrame(data={'alpha':alpha},index=df.index[window-1:])
    beta = pd.DataFrame(data={'beta':beta},index=df.index[window-1:])
    new_df = pd.concat([df,alpha,beta],axis=1)
    new_df['residual'] = new_df['{}'.format(target[1])] - new_df['alpha'] + new_df['beta']*new_df['{}'.format(target[0])]
    return new_df
rollingreg(df)[57:62]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0386.HK</th>
      <th>0857.HK</th>
      <th>alpha</th>
      <th>beta</th>
      <th>residual</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2015-11-27</th>
      <td>4.470203</td>
      <td>5.459994</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2015-11-30</th>
      <td>4.460812</td>
      <td>5.430428</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2015-12-01</th>
      <td>4.582897</td>
      <td>5.568406</td>
      <td>1.546656</td>
      <td>0.864888</td>
      <td>7.985442</td>
    </tr>
    <tr>
      <th>2015-12-02</th>
      <td>4.601679</td>
      <td>5.617684</td>
      <td>1.451807</td>
      <td>0.883489</td>
      <td>8.231409</td>
    </tr>
    <tr>
      <th>2015-12-03</th>
      <td>4.526550</td>
      <td>5.588117</td>
      <td>1.352018</td>
      <td>0.903162</td>
      <td>8.324308</td>
    </tr>
  </tbody>
</table>
</div>


<br/> 

Here is a function that does rolling regression. Since pandas rolling regression function only returns beta, I defined my own one that fully returns alpha, beta and residual. Because I defined a rolling window of 60 days, the regression parameters only have values starting from the 60th row.



```python
def step1(df=df):
    new_df = df.pct_change().iloc[1:]
    return rollingreg(new_df)
step1()[57:62]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0386.HK</th>
      <th>0857.HK</th>
      <th>alpha</th>
      <th>beta</th>
      <th>residual</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2015-11-30</th>
      <td>-0.002101</td>
      <td>-0.005415</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2015-12-01</th>
      <td>0.027368</td>
      <td>0.025408</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2015-12-02</th>
      <td>0.004098</td>
      <td>0.008850</td>
      <td>-0.000668</td>
      <td>0.907991</td>
      <td>0.013239</td>
    </tr>
    <tr>
      <th>2015-12-03</th>
      <td>-0.016326</td>
      <td>-0.005263</td>
      <td>-0.000581</td>
      <td>0.905394</td>
      <td>-0.019464</td>
    </tr>
    <tr>
      <th>2015-12-04</th>
      <td>-0.002075</td>
      <td>-0.022928</td>
      <td>-0.000675</td>
      <td>0.922321</td>
      <td>-0.024166</td>
    </tr>
  </tbody>
</table>
</div>



<br/> 

Here we first convert the stock-price series into its percentage change. From the regression parameters, we verify that  `alpha` is very close to zero, so we can ignore it for now. `beta` would be used for robustness checks. Since the model expects a constant beta, stock pairs with an versatile beta within our targeted trading time span would impose a bigger risk on the trades. The paper written by Y Chen mentions that a stable rolling beta for 5 days would be a baseline signal. In reality, this would be case dependent.


```python

import matplotlib.pyplot as plt
df = step1()
df.plot(x=df.index,y='beta')
plt.show()

```


![png]({{ site.baseurl }}/assets/media/PairTrade/2/output_9_0.png)


Unfortunately, the stock pair 0386.HK and 0857.HK does not demonstrate a stable beta in the past two years. especially in recent months. It oscillated from 0.7 to 1 with a sudden drop of 0.5 in the past six months. However, for demonstration purpose I would continue to use this as an example. Now, after the first regression is done, the rolling sum of residual term would be used for another round of regression against itself with one day delay ~ (R = a + b x R(delay=1) + error). This is equivalent to autoregression with degree 1 / AR(1). 


```python
import numpy as np
def step2(df=df):
    df2 = step1(df)
    #df has alpha,beta and residual
    new_df = pd.DataFrame()
    new_df['sum_residual'] = df2['residual'].rolling(window=window).sum()
    new_df['sum_residual_delay'] = new_df['sum_residual'].shift(1) #regressor ~ X
    target = ['sum_residual_delay','sum_residual'] #[X,Y]
    new_df = rollingreg(new_df,target=target) #rerun rolling regression with updated alpha and beta and residual
    new_df['var'] = new_df['residual'].rolling(window=window).std(skipna=True)**2
    new_df['equsigma'] = new_df['var']/(1 - new_df['beta']**2)
    new_df['mu'] = new_df['alpha']/(1- new_df['beta'])
    #new_df['speed'] = -np.log(new_df['beta'])*252
    #new_df['std'] = np.sqrt(new_df['equsigma']*2*new_df['speed'])
    return new_df
step2()[235:240]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sum_residual</th>
      <th>sum_residual_delay</th>
      <th>alpha</th>
      <th>beta</th>
      <th>residual</th>
      <th>var</th>
      <th>equsigma</th>
      <th>mu</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2016-08-19</th>
      <td>0.163828</td>
      <td>0.242463</td>
      <td>0.050941</td>
      <td>0.721709</td>
      <td>0.287875</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.183050</td>
    </tr>
    <tr>
      <th>2016-08-22</th>
      <td>0.141429</td>
      <td>0.163828</td>
      <td>0.047130</td>
      <td>0.741717</td>
      <td>0.215813</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0.182475</td>
    </tr>
    <tr>
      <th>2016-08-23</th>
      <td>0.138764</td>
      <td>0.141429</td>
      <td>0.050200</td>
      <td>0.717386</td>
      <td>0.190023</td>
      <td>0.014419</td>
      <td>0.029708</td>
      <td>0.177627</td>
    </tr>
    <tr>
      <th>2016-08-24</th>
      <td>0.130538</td>
      <td>0.138764</td>
      <td>0.046922</td>
      <td>0.735626</td>
      <td>0.185694</td>
      <td>0.013847</td>
      <td>0.030176</td>
      <td>0.177485</td>
    </tr>
    <tr>
      <th>2016-08-25</th>
      <td>0.117359</td>
      <td>0.130538</td>
      <td>0.043429</td>
      <td>0.757017</td>
      <td>0.172749</td>
      <td>0.013978</td>
      <td>0.032742</td>
      <td>0.178732</td>
    </tr>
  </tbody>
</table>
</div>


<br/> 


First, `sum_residual` is the 60-day rolling sum of the residuals we get from the first regression of stock price(percentage change). It would be the `Y` in the coming regression. 

Column `sum_residual_delay` is `sum_residual` with one-day delay. It would be the  `X` in the coming regression.

Column alpha,beta and residual are the regression parameters resulted from regressing `sum_residual` on `sum_residual_delay`. 

`mu` and `equisignma` are directly related to our signal generation:

`mu`: The mean of residual we would expect.
`equisignma`: The standard deviation of mu we would expect 

Now, whenever the residual term exceeds the level of `mu` plus some degree of `equisignma` (z-score depending on the risk level), we can short sell the spread. Recall how regression is done(Y=a+bx+error), concretely short selling the spread means short Y long X(Y-X+). On the other hand, when the residual term plunges through the level of `mu` minus some degree of `equisignma`, we long the spread, long Y short X (Y+X-).



Picking a z-score of 1, Let's see how the pnl performs:



```python

import matplotlib.pyplot as plt
def pnl(new_df):
    ###create a signal for long/short/hold
    def signal_gen(x):
        if x['residual'] - x['mu'] > x['equsigma']:
            return 1
        elif x['residual'] - x['mu'] < x['equsigma'] and x['residual'] - x['mu'] > -x['equsigma']:
            return 0
        elif x['residual'] - x['mu'] < -x['equsigma']:
            return -1
        else:
            return 2
        
    openprice['signal'] = new_df.apply(signal_gen,axis=1) #signal -1/0/1, shifted
    holding = 0  #current status
    pnL_histroy = []  ##pnl 
    entry = [] #store X,Y and their price ratio for contract entry
    shift = openprice.index[1:]  #for opening contract
    
    for i in range(1,new_df.shape[0]):
        if openprice.loc[shift[i-1],'signal'] == 1: #excessive spread
            if holding == 0:  #if not holding any contract,
                # because of excessive spread, short Y long X
                #entry [X,Y, Y/X]
                entry = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]],openprice.loc[shift[i],target[1]]/openprice.loc[shift[i],target[0]]] #target[0] is X,1 is Y
                holding = -1 #now hold a short contract
                pnL_histroy.append(0) #no pnl at this day
            elif holding == -1:  #if holding an existing short contract
                #update pnl
                pnl = (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] + openprice.loc[shift[i],target[1]] - entry[1] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                #update entry to price of current day, keep entry ratio
                entry[:2] = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]]] #target[0] is X,1 is Y
            elif holding == 1: #if holding an existing long contract
                #update pnl
                pnl = openprice.loc[shift[i],target[1]] - entry[1] + (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                ##now open a short contract
                holding = -1
                entry = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]],openprice.loc[shift[i],target[1]]/openprice.loc[shift[i],target[0]]] #target[0] is X,1 is Y
            else:
                print('error for holding')
                
        elif openprice.loc[shift[i-1],'signal'] == -1: 
            if holding == -1: 
                pnl = (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] + openprice.loc[shift[i],target[1]] - entry[1] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                holding = 1
                entry = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]],openprice.loc[shift[i],target[1]]/openprice.loc[shift[i],target[0]]] #target[0] is X,1 is Y
            elif holding == 0:
                holding = 1
                entry = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]],openprice.loc[shift[i],target[1]]/openprice.loc[shift[i],target[0]]] #target[0] is X,1 is Y
                pnL_histroy.append(0)
            elif holding == 1:
                pnl = openprice.loc[shift[i],target[1]] - entry[1] + (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                entry[:2] = [openprice.loc[shift[i],target[0]],openprice.loc[shift[i],target[1]]] #target[0] is X,1 is Y
            else:
                print('error for holding')
                
        elif openprice.loc[shift[i-1],'signal'] == 0:
            if holding == 0:
                pnL_histroy.append(0)
            elif holding == 1:
                pnl = openprice.loc[shift[i],target[1]] - entry[1] + (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                holding = 0
            elif holding == -1:
                pnl = (entry[0] - openprice.loc[shift[i],target[0]])*entry[2] + openprice.loc[shift[i],target[1]] - entry[1] #target[0] is X,1 is Y
                pnL_histroy.append(pnl)
                holding = 0
            else:
                print('error for holding')
        else: ###sginal 2 or na are not in the timeframe of analysis, ignore
            pass  

    y = shift.to_datetime()[-len(pnL_histroy):]
    pnl, = plt.plot(y,pnL_histroy,label='pnl')
    cumsum, = plt.plot(y,np.cumsum(pnL_histroy),label='cumulative pnl')
    plt.legend(handles=[pnl,cumsum])
    plt.show()
    result = {'profit':sum(pnL_histroy),
    'win ratio':sum([1 for i in pnL_histroy if i>0])/(sum([1 for i in pnL_histroy if i<0])+sum([1 for i in pnL_histroy if i>0])),
    'maximum day gain':max(pnL_histroy),'maximum day loss':min(pnL_histroy)}
    return result
pnl(step2())
```


![png]({{ site.baseurl }}/assets/media/PairTrade/2/output_14_0.png)





    {'maximum day gain': 0.29080789946140051,
     'maximum day loss': -0.19394827586206809,
     'profit': 0.039680522576772459,
     'win ratio': 0.43506493506493504}




After the PnL simulation, the pair performs quite poorly for trading. It echoes the fact that the pair does not possess a stable beta.



