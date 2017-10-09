---
layout: post
header:
title: Web Scraping IV - Scrapy and Sentiment Analysis
categories: [Python, Web Scraping, Sentiment Analysis]
tags: [Python, Web Scraping, TextBlob, Scrapy, Bash]
fullview: true
comments: true
---
**Project Description:**

> **Scrap forex news specific to each currency pair in the last 24 hours on FXStreet at 7am each day, then calculate an average sentiment score**

**Use of Tools:**
- **Scrapy**: a web crawler framework and data extraction API
- **Splash & scrapy_splash**: a javascript-rending tool, so that scrapy spider can crawl responses with javascript.
- **TextBlob**: a light-weight wrapper of NLTK, which is a full-fledged, comprehensive neutral language processing library.

**Installation:**
I don't think I can explain better than the official documentations:
- [Scrapy](https://scrapy.org/download/)
- [Splash & scrapy_splash](https://github.com/scrapy-plugins/scrapy-splash)
- [TextBlob](https://textblob.readthedocs.io/en/dev/)


**The Evaluation Steps:**

1. Since I would like to get news for each pair, so I have first gone to the news page of [each curreny pair](https://www.fxstreet.com/news/latest?dFR%5BCategory%5D%5B0%5D=News&dFR%5BTags%5D%5B0%5D=EURUSD).

2. As soon as I started turning over to the new page, I noticed that FXStreet allows custom pagination in the url, so I could specify a much larger number of news on a page than what I would need, eg: 50, to avoid navigating to the next page, which would involve additional lines. The [url](https://www.fxstreet.com/news/latest?q=&hPP=50&idx=FxsIndexPro&p=0&dFR%5BCategory%5D%5B0%5D=News&dFR%5BTags%5D%5B0%5D=EURUSD) here is structured differently, a parameter 'PP' is added, which determines pagination.

3. After inspecting the page elements, you can see that the main table with all the news is rendered by javascript in the browser. In that case, the direct response from the url does not contain the information I need. So I looked for a way to crawl javascript content.

4. It is when Splash comes into place. Once Splash is up and running through docker, I managed to get the expected return from the page.

5. A tip: For testing, you can download the txt file from Splash opened in a browser at  port 8050 (Default) `localhost:8050`. Then run the file in scrapy shell `scrapy shell file.html` after converting the file into html format. From there you can experiement the methods associated with the response and selectors (a scrapy object).
    

**Preliminary Steps:**

1. Create a project file by :
`$ scrapy startproject fxnews`

2. Make the necessary change mentioned in the documentation of Splash in ``settings.py``. Here is the setting that I added :


```python

SPLASH_URL = 'http://localhost:8050/'
DOWNLOADER_MIDDLEWARES = {
    'scrapy_splash.SplashCookiesMiddleware': 723,
    'scrapy_splash.SplashMiddleware': 725,
    'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware': 810,
}
SPIDER_MIDDLEWARES = {
    'scrapy_splash.SplashDeduplicateArgsMiddleware': 100,
}
DUPEFILTER_CLASS = 'scrapy_splash.SplashAwareDupeFilter'
HTTPCACHE_STORAGE = 'scrapy_splash.SplashAwareFSCacheStorage'
SPLASH_COOKIES_DEBUG = True
SPLASH_LOG_400 = False

```

The entire crawler under directory ``spiders``: 


```python

from datetime import datetime
import re
import scrapy
from scrapy_splash import SplashRequest

class QuotesSpider(scrapy.Spider):
    name = 'fxnews'
    # days would be a system arguement, which is fed from a bash script
    # scrapy allow system arguements without use of sys.parse eg:
    # $ scrapy crawl fxnews -a days=5` 

    def __init__(self,days):
        self.days = int(days)
        self.pairs = ['EURUSD','USDJPY','EURGBP']
        self.urls = [
        'https://www.fxstreet.com/news/latest?q=&hPP=50&idx=FxsIndexPro&p=0&dFR%5BCategory%5D%5B0%5D=News&dFR%5BTags%5D%5B0%5D={}'.format(pair)
        for pair in self.pairs]
    
    # for each currency pair, make a Splash call so the response 
    # would render the javascript in the page.
    # 5 second is needed for safety to make sure the javascript is rendered.
    def start_requests(self):
        for url in self.urls:
            yield SplashRequest(url=url ,callback=self.parse
            ,args={'wait': 5})
    
    # function to be called on news of each pair
    def parse(self,response):
        #narrow down to news within a defined period(1,2,3 days..)
        current_year = datetime.now().year
        time_list = response.css('address.fxs_entry_metaInfo time::text').extract()
        time_list_updated = [datetime.strptime(time + str(current_year), '%b %d, %H:%M GMT%Y') for time in time_list]
        latest_time = [i for i in time_list_updated if (datetime.now() - i).total_seconds() < 86400 * self.days]
        num_eligible = len(latest_time)
        
        # for each piece of news, make a scrapy call and yield its news body by function - get_news_content
        hrefs = response.css('h4.fxs_headline_tiny a::attr(href)').extract()[:num_eligible]
        for href in hrefs:
            news_body = scrapy.Request(url=href,callback=self.get_news_content)
            yield news_body
    
    # for each url on news, return {keywords, title, content}
    # eg: {'keywords':'EURUSD','title':'EURUSD up,..','content':'bababa...'}
    def get_news_content(self,response):
        selectors = response.xpath('//script[contains(@type,"application/ld+json")]')

        titles = [re.search('"name": "(.*)',json_body) for selector in selectors
        for json_body in selector.css('script').extract()]
        title = [i for i in titles if i is not None][0].group(1)

        all_content = [re.search('"articleBody" : "(.*)',json_body) for selector in selectors
        for json_body in selector.css('script').extract()]
        content = [i for i in all_content if i is not None][0].group(1)

        keys = [re.search('"keywords": ".*?([A-Z]{6})',json_body) for selector in selectors
        for json_body in selector.css('script').extract()]
        key = [i for i in keys if i is not None][0].group(1)

        yield {
        'keywords':key,
        'title':title,
        'content':content
        }

```

Once the crawler is written, it can be called by outputting its result to a json file :

```sh
$ scrapy crawl fxnews -o news.json
```

From there sentiment analysis can be conducted.
TextBlob contains an out-of-the-box function `TextBlob('text').sentiment()` that returns a numpy array `np.array(sentiment_score, subjectivity)`.

The sentiment score ranges from **-1 to 1**, while objectivity ranges from **0 to 1** (total objectivity = 0; total subjectivity = 1). 

For simplicity, I evaulated the score by multiplying both elementes to come up with a score for one piece of news, then get the aggregate average of all news concerning one currency pair.


```python

from textblob import TextBlob
import json
import numpy as np
from datetime import datetime

def sentiment_score():
    # load the json containing the news collected from the crawler
    try:
        all_news = json.load(open('news.json'))
        # all pairs
        pairs = list(set([i['keywords'] for i in all_news]))
    except FileNotFoundError:
        print('File not found')
        return None
    except json.decoder.JSONDecodeError:
        print('No news are found')
        return None

    # append the title and content for sentiment analysis
    for pair in pairs:
        news = [piece['title'] + piece['content'] for piece in all_news if piece['keywords']==pair]
        sentiment_tuple = [TextBlob(news[i]).sentiment for i in range(len(news))]
        score = np.mean([i[0]*i[1] for i in sentiment_tuple])
        print(datetime.now(),' Total News:', len(news))
        print({pair: score},'\n')

if __name__ == '__main__':
    sentiment_score()
    
```

Finally, make call to both scripts by a simple bash ``execution.sh``:
    

```bash
#!/bin/bash
# specify the duration of news to collect
# default to 1 day
if [ ! -z $1 ];
then
  days=$1
else
  days=1
fi

#remove previous collected news before a new round
rm news.json
scrapy crawl fxnews -o news.json -a days=$days
python3 sentiment.py

```

Finally, simply run :

```bash
$ bash execution.sh
```

Then the result would be like this:

2017-10-08 21:10:54.713816  Total News: 8

{'EURGBP': 0.0062293261135899728} 

2017-10-08 21:10:54.839823  Total News: 50

{'EURUSD': 0.014449971221041913} 

2017-10-08 21:10:54.897309  Total News: 27

{'USDJPY': 0.021803648466084024}


To schedule the task to be performed at certain time every day, eg: 7am, simple use cron :

```bash
 $ contab -e
 $ 0 7 * * * /bin/bash /path/to/execution.sh > output.txt
```
