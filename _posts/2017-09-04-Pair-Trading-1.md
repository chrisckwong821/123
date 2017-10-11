---
layout: post
header-img: "img/fractal.jpg"
title: Pair Trading - I - DBSCAN on HK Stocks
subtitle: Selecting Stock Pair with a Click
categories: [Machine Learning, Financial Trading]
tags: [Python, Machine Learning, Pair Trading]
fullview: true
comments: true
---

I was inspired by the [Pairs Trading with Machine Learning](https://www.quantopian.com/posts/pairs-trading-with-machine-learning) on Quantopian, where the author Mr. Jonathan Larkin applies DBSCAN, a clustering technique to select stock pairs that behave similarly. Using a programmatic approachFrom, he managed to boil down to 90 pairs from an universe of 1500 stocks, or 1 million pairs.

AS of 3-Sep-2017, There are 934 stocks which are eligible for short sell in HK stock exchange. The basket is quite big. However, there are not much free online resources or toolkit on HK stocks that come close to what quantopian does on the US stocks. Most of the brokeage firm, data vendors only provide free analytical report, or simply query searching, but not an API or a system which allows reproducible analysis.

Therefore, I would like to conduct an analysis on pair trading specific to Hong Kong stocks.

If you want to go straight to see which pairs of stocks have the closest relationship in price movement. You may go straight to the bottom of the article.

Before the implementation, it is good to motivate the topic by stating what the current situation is and what I would like to do.


>Among the 934 stocks eligible for short sell, only the most liquid and 
>hot equities are the more usually target of short-sell. The blue-chip >stocks have a short sell ratio up to 30% (eg: Tencent), while the second-tier stocks only have a few percentages. Clearly part of the reason for the discrepancy is that investors are worried about the short squeeze or noisy trading behaviors due to smaller liquidity on those stocks. Because of this, at aggregate level, there may be a gap where institutional investors can bridge by creating a pair trading >portfolio with specific strategy and breadth on stock selection.

**First Step:** Donwload the stock price of the 934 stocks.

The ticker of the list can be found on the [HKEx](https://www.hkex.com.hk/eng/market/sec_tradinfo/dslist.htm).

After downloading the ticker, I have written a crawler to get the stock prices from Yahoo Finance.To spare you from the trouble, I have uploaded the zip file containing all the stock prices over [here](https://github.com/chrisckwong821/chrisckwong821.github.io/blob/master/assets/Reference/StockData.zip)

But if you are interested in the crawler as well, here is the code:



```python
import pandas as pd
import time
import re
from urllib.request import urlopen, Request, URLError
import calendar
import datetime

def crawler():
    #read in stock code
    listofstocks = list(pd.read_csv('ds_list20170831.csv')['Stock Code'])
    ticker = []
    #format it for crawler
    for i in listofstocks:
        i = str(i)
        if len(i) < 4:
            i = '0'*(4-len(i)) + (i) + '.HK'
        else:
            i = i + '.HK'
        ticker.append(i)
    return ticker

crumble_link = 'https://finance.yahoo.com/quote/{0}/history?p={0}'
crumble_regex = r'CrumbStore":{"crumb":"(.*?)"}'
cookie_regex = r'set-cookie: (.*?); '
quote_link = 'https://query1.finance.yahoo.com/v7/finance/download/{}?period1={}&period2={}&interval=1d&events={}&crumb={}'
##usage: download_quote(symbol, date_from, date_to, events).decode('utf-8')

def get_crumble_and_cookie(symbol):
    link = crumble_link.format(symbol)
    response = urlopen(link) #the response
    match = re.search(cookie_regex, str(response.info())) #get the cookie
    cookie_str = match.group(1)
    text = response.read().decode("utf-8")
    match = re.search(crumble_regex, text) #get the crumble
    crumble_str = match.group(1)
    return crumble_str , cookie_str #return both

def download_quote(symbol, date_from, date_to,events):
    ####### this part convert the date into format intended by Yahoo Finance
    time_stamp_from = calendar.timegm(datetime.datetime.strptime(date_from, "%Y-%m-%d").timetuple())
    next_day = datetime.datetime.strptime(date_to, "%Y-%m-%d") + datetime.timedelta(days=1)
    time_stamp_to = calendar.timegm(next_day.timetuple())
    ########
    attempts = 0
    while attempts < 5:
        crumble_str, cookie_str = get_crumble_and_cookie(symbol)
        link = quote_link.format(symbol, time_stamp_from, time_stamp_to, events,crumble_str)
        ##### request
        r = Request(link, headers={'Cookie': cookie_str})
        try:
            #####store the response into a csv file
            response = urlopen(r)
            text = response.read()
            with open('{}.csv'.format(symbol), 'wb') as f:
                f.write(text)
            print("{} downloaded".format(symbol))
            return b''
        except URLError:
            print ("{} failed at attempt # {}".format(symbol, attempts))
            attempts += 1
            time.sleep(2*attempts)
    return b''
    
if __name__ == '__main__':
    start_date = '2015-09-03'
    end_date = '2017-09-03'
    event = 'history'
    ticker = crawler()
    for i in ticker:
        download_quote(i,start_date,end_date,event)

```

`crawler` is the function which standardize the stock code downloaded from HKEx to the format used by Yahoo Finance. `get_crumble_and_cookie` would get crumble(like an API key, unique to each cookie) and cookie by making a dummy call to yahoo finance. `download_quote` is the function which downloads the csv file.


For pair trading, the day over day percentage change is what matters:


```python
import pandas as pd
import os.path

def crawler():
    #read in stock code
    listofstocks = list(pd.read_csv('ds_list20170831.csv')['Stock Code'])
    ticker = []
    #format it for crawler
    for i in listofstocks:
        i = str(i)
        if len(i) < 4:
            i = '0'*(4-len(i)) + (i) + '.HK'
        else:
            i = i + '.HK'
        ticker.append(i)
    return ticker

def get_dod():
    path = os.path.join(os.getcwd(),"StockData/")
    df = pd.DataFrame()
    for i in crawler():
        try:
            DOD = pd.read_csv( path + '{}.csv'.format(i))['Adj Close']
            DOD.replace(to_replace='null',method='ffill',inplace=True)
            DOD = pd.to_numeric(DOD)
            DOD = DOD.pct_change()[1:]
            df = pd.concat([df,DOD.rename('{}'.format(i))],axis=1)
        except:
            #print(i+' fails to be loaded') (for control)
            pass
    df.index = pd.to_datetime(pd.read_csv(path + '0001.HK.csv',error_bad_lines=False).pop('Date')[1:])
    return df.dropna(axis=1)

if __name__ == '__main__':
    x = get_dod()
    print(x.shape)

```

    (493, 638)
    Stock without any missing data: 638

**Data Descriptions**
---

There are 18 tickers which cannot be loaded. Among them, 
['3026.HK','4362.HK','4363.HK','9081.HK'] 
are absence from the list at all, meaning that no file is actually downloaded from Yahoo Finance. The other 14 are only listed within these two years and have a varying length of trading time. Since they only account for a small number, we would excluded them from the basket for now to preserve model integrity. 

For the remaining 916 stocks, a lot of them has null values throughout the columns, or consists of many NaN values. A closer look into the data, shows that some of them dont have the time series data up to two years.

So now we are left with 638 stocks, around 2/3 of the original dataset. It is quite a bit of loss but still acceptable considering the source of data is from web only.
---

**Dimensionality Reduction:**

We may ponder the necessarity of dimensionality reduction, for a dataframe as small as (493, 638), certainly the computer can manage the computation of calculating each point's euclidan distance(or other distance matrics) for clustering to work, but feeding each day's percentage return into the clustering algorithm may lead to overfitting, in the sense that most stocks may be sparse from each other, left the dataset with one big central cluster only. I have tried with different combinations of parameters (the radius search of one point/eps) to find the most distinct clustering, however it is still not as good as the one with a reduced matrix.

**Implementation without Dimensionality Reduction**


```python
from sklearn.decomposition import PCA
from sklearn.cluster import DBSCAN
import matplotlib.pyplot as plt

def dod_analysis(eps,n_components='NA'):
    returns = get_dod()
    #feeding all changes 
    if n_components == 'NA':
        X = returns.as_matrix().transpose()
    else:
        pca = PCA(n_components=n_components)
        X = pca.fit(returns).components_.T  
    ###clustering
    clf = DBSCAN(eps=eps, min_samples=2)
    clf.fit(X)
    labels = clf.labels_
    
    n_clusters_ = len(set(labels)) - (1 if -1 in labels else 0)
    print('cluster found: ', n_clusters_)
    clustered = clf.labels_
    clustered_series = pd.Series(index=returns.columns, data=clustered.flatten())
    clustered_series_all = pd.Series(index=returns.columns, data=clustered.flatten())
    clustered_series = clustered_series[clustered_series != -1]
    counts = clustered_series.value_counts()
    plt.barh(
    range(len(clustered_series.value_counts())),
    clustered_series.value_counts())
    plt.title('Cluster Member Counts')
    plt.xlabel('Stocks in Cluster')
    plt.ylabel('Cluster Number')
    plt.show()
    
dod_analysis(eps=0.25)
```

    cluster found:  4

![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_10_1.png)


The opposite end of not doing any dimensionality reduction at all, is to condense the data so much that even distant stocks cluster together.


```python

dod_analysis(eps=0.005,n_components=10)

```

    cluster found:  18



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_12_1.png)



For this set of parameters, 18 clusters are found. Apparaently this is much more than the above one where no dimensionality reduction is done. So how can we strike the balance between these two? I think the best way is to constrain the number of pairs we would like to look into, and then reduce the level of reduction (use more n_components), in order to find the sweet spot.

For me, I dont want to look into more than three or four clusters in-depth. So after a few trial-and-error:



```python

dod_analysis(eps=0.1,n_components=200)

```

    cluster found:  3



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_14_1.png)



With 3 clusters of stocks (3,4,4), There are only three graphs and 6+6+3 = 15 pairs only.
This is exactly the size of space I intended.

Now let's get into the content of these spaces:

```python
import numpy as np

def dod_plot(eps,n_components):
    returns = get_dod()
    if n_components == 'NA':
        X = returns.as_matrix().transpose()
    else:
        pca = PCA(n_components=n_components)
        X = pca.fit(returns).components_.T  
    clf = DBSCAN(eps=eps, min_samples=2)
    clf.fit(X)
    labels = clf.labels_
    clustered = clf.labels_
    clustered_series = pd.Series(index=returns.columns, data=clustered.flatten())
    clustered_series_all = pd.Series(index=returns.columns, data=clustered.flatten())
    clustered_series = clustered_series[clustered_series != -1]
    counts = clustered_series.value_counts()

    # let's visualize some clusters
    cluster_vis_list = list(counts.index)[::-1]
    # plot a handful of the smallest clusters
    for clust in cluster_vis_list[0:min(len(cluster_vis_list), 3)]:
        tickers = list(clustered_series[clustered_series==clust].index)
        data = np.exp(np.log1p(returns[tickers]).cumsum())
        print(tickers)
       #means = np.log(returns[tickers].mean())
       # data = np.log(returns[tickers]).sub(means)
        data.plot(title='Stock Time Series for Cluster %d' % clust)
        plt.show()

dod_plot(eps=0.1,n_components=200)

```

    ['2822.HK', '2823.HK', '82822.HK']


![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_16_1.png)


    ['0247.HK', '2666.HK', '2819.HK', '2821.HK']



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_16_3.png)


    ['2800.HK', '3007.HK', '3040.HK', '3055.HK']



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_16_5.png)



For ['2822.HK', '2823.HK', '82822.HK']
---
All three are China ETF, giving them a solid ground on fundamentals for pair trading. However, it also implies a small room for arbitrage as their price tend to behave very closely to one another. Also a lot of institutional investors are already monitoring them.

For ['0247.HK', '2666.HK', '2819.HK', '2821.HK']:
---
Omitting the two non-performaing stocks, **2819/2821** have the same underlyings : **Markit iBoxx ABF Pan-Asia Index**. However they exhibited a diverging discrepancy since the second quarter last year, all the way until the end of last year. This is indeed a good case of pair trading had we noticed the opportunity. Now the 2821 is trading slightly at premium than 2819, probably can keep an eye on this pair.

For ['2800.HK', '3007.HK', '3040.HK', '3055.HK']:
---
**2800/3007** are HSI/China index funds respectively. From the price perspectively, they behave quite nicely since they cross one another from time to time. However 3007 is not quite liquid as an ETF, so in reality it is not practical to capture the spread.
**3040/3055** are China ETF as well. both are not liquid.



Out of curiosity, I want to look into the three pairs that appear when the dataset undergoes no dimensionality reduction:



```python

dod_plot(eps=0.25,n_components='NA')

```

    ['0386.HK', '0857.HK']



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_19_1.png)


    ['0737.HK', '80737.HK']



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_19_3.png)


    ['6030.HK', '6837.HK']



![png]({{ site.baseurl }}/assets/media/PairTrade/1/output_19_5.png)


['0386.HK', '0857.HK']:
---
both are oil/energy SEOs, sensible with the perspective of fundamental factor. Now their price are exhibiting a diverging behavior. I am not an analyst of corporate finance but the result obviously shows that it is worth looking into their status to see if a pair trade can be established.

['0737.HK', '80737.HK']:
---
Non-liquid stocks with the same underlying infrastructure investment.

['6030.HK', '6837.HK']:
---
Both are Chinese financial stocks with enough liquidity. They also exhibits similar price movements in the past two years with possibly a moving drift. Worth looking into as well.


__Summary:__

A clustering method, DBSCAN with dimensionality reduction is implemented to select potential pairs for trading in HKEx, based on stocks' geometric return/ Day-over-day percentage change. 

For the next episode, I would look into the mean-reverting models that can be used to imply a moving/drifting mean for the spread between stocks pairs.



Again, send me an email or leave some comments if you like the article. Any constructive feedback is very much welcome.

