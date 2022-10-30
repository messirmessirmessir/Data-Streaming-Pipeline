# Building a Highly Scalable Data Streaming Pipeline in Python | by Muhammad Haseeb | Geek Culture | Medium
A step by step guide to building a highly scalable data streaming pipeline in Python
------------------------------------------------------------------------------------

![](https://miro.medium.com/max/1400/0*eKutYzBCSA79e8P4)

Photo by [JJ Ying](https://unsplash.com/@jjying?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Python has shaped itself as a language for data intensive jobs. We are seeing it everywhere just because it’s really fast to prototype in Python and people are loving it due to its easy syntax, that wave landed in the data industry too. Data engineers and data scientists also started using it in their data intensive jobs. In this story, we are going to build a very simple and highly scalable data streaming pipeline using Python.

> Data streaming is the process of transmitting a continuous flow of data.

![](https://miro.medium.com/max/1242/1*HpfNnta4wcwuPPDWrFkv4Q.png)

Photo by [Author](http://haseeb-writes.medium.com/)

Now we know on one side of the pipeline we would have some or at least one data producer who would be continuously producing data and on the other side, we would have some or at least one data consumer who would be consuming that data continuously.

Architecture
------------

First thing is to design a scalable and flexible architecture that justifies the claim. We’ll be using [Redis](https://redis.io/) as the data pipeline and for the sake of this story, we’ll be using a very simple data scraping microservice using [Scrapy](https://scrapy.org/) independently as a data producer and a separate microservice as a data consumer.

![](https://miro.medium.com/max/1400/1*9m4bX85qMH3jAeimuDR_kQ.png)

Photo by [Author](http://haseeb-writes.medium.com/)

**Building the data Producer**
------------------------------

The first we need is to build a simple Python project with an activated virtual environment. For this specific story, we are going to use Scrapy’s official [tutorial](https://docs.scrapy.org/en/latest/intro/tutorial.html). We need to run the command given below to create an empty Scrapy project.

`scrapy startproject producer`

This would create a directory structure like shown in the image below

![](https://miro.medium.com/max/1400/1*WKIuWRVV7XC-XREplnjIXA.png)

Photo by [Author](http://haseeb-writes.medium.com/)

Now we need to create a spider that can actually scrape the data from somewhere. Let’s create a new file in the spiders directory `quotes_spider.py` and add the code given below to it.
```py
import scrapy

class QuotesSpider(scrapy.Spider):  
    name = "quotes"

    def start\_requests(self):  
        urls = \[  
            'https://quotes.toscrape.com/page/1/',  
            'https://quotes.toscrape.com/page/2/',  
        \]  
        for url in urls:  
            yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response, \*\*kwargs):  
        for quote in response.css('.quote .text::text').getall():  
            yield {  
                'quote': quote  
            }
```
This specific code would visit page1 and page2 on the website and would extract titles of quotes. To run this spider we need to type `cd producer` to go into the scrapy project directory and run spider with `scrapy crawl quotes -o ouput.json` . `-o` would point all the yielded data to `output.json` file.

Our data producer side is ready now but we need to put that data into a data pipeline instead of in a file. Before putting data into a data pipeline we need to build a data pipeline before.

Building a data pipeline
------------------------

Firstly we need to install Redis on our system, to do that we need to follow the official installation [guide](https://redis.io/docs/getting-started/installation/) of Redis. After installing the Redis and running it should show something like the below.

![](https://miro.medium.com/max/1400/1*WQBfhl15T0msGCLm051EEg.png)

Photo by [Author](http://haseeb-writes.medium.com/)

Now we need to create wrappers around Redis functions to make them more humane. Let’s start with creating a directory in `root` with a name `pipeline` and create a new file `reid_client.py` in this directory.

![](https://miro.medium.com/max/1400/1*-vYLgVE5i-npPDVp6Vnldw.png)

Photo by [Author](http://haseeb-writes.medium.com/)

Now we need to add the code given below to our `redis-client.py` file. The code is self-explanatory we created a data getter and data setter. Both deal with JSON data as Redis can only store string data and store string data we need to JSONIFY it.

```py
import json

import redis

class RedisClient:  
    _"""  
    Custom Redis client with all the wrapper funtions. This client implements FIFO for pipeline.  
    """_    connection = redis.Redis(host='localhost', port=6379, db=0)  
    key = 'DATA-PIPELINE-KEY'

    def _convert_data_to_json(self, data):  
        try:  
            return json.dumps(data)  
        except Exception as e:  
            print(f'Failed to convert data into json with error: {e}')  
            raise e

    def \_convert\_data\_from\_json(self, data):  
        try:  
            return json.loads(data)  
        except Exception as e:  
            print(f'Failed to convert data from json to dict with error: {e}')  
            raise e

    def send\_data\_to\_pipeline(self, data):  
        data = self.\_convert\_data\_to\_json(data)  
        self.connection.lpush(self.key, data)

    def get\_data\_from\_pipeline(self):  
        try:  
            data = self.connection.rpop(self.key)  
            return self.\_convert\_data\_from\_json(data)  
        except Exception as e:  
            print(f'Failed to get more data from pipeline with error: {e}')
```

Putting data into the pipeline from the producer
------------------------------------------------

Now as we have created a pipeline we can start putting our data into that from the data producer side. For that, we need to create a pipeline in scrapy which adds every scraped item to Redis and we consume it later. Simply add the code below to your `pipelines.py` the file of scrapy project.

```
from pipeline.redis\_client import RedisClient

class ProducerPipeline:  
    redis\_client = RedisClient()

    def process\_item(self, item, spider):  
        self.redis\_client.send\_data\_to\_pipeline(item)  
        return item
```

We would also need to enable this pipeline in our scrapy project, for that we would need to uncomment the lines given below in `settings.py` of scrapy project.

```
ITEM\_PIPELINES = {  
   'producer.pipelines.ProducerPipeline': 300,  
}
```
This would start sending the data to Redis and to verify we can check our pipeline with `redis-cli` and type `LLEN 'DATA-PIPELINE-KEY’` to see the number of quotes in the data pipeline.

Building the consumer and consuming data
----------------------------------------

As we’ve built a pipeline and a producer which can keep putting data to the pipeline independent of data consumption we are more than halfway through all we need to get data from the data pipeline and consume it according to our needs to call it a project.

Let’s create a new directory in `root` , name it `consumer` and create a new file with the name `quotes_consumer.py` in it.

![](https://miro.medium.com/max/1400/1*YIEVGUjLSiTyQDuwgl_xpQ.png)

Photo by [Author](http://haseeb-writes.medium.com/)

Add the code given below to `quotes_consumer.py` file. Code checks for new data if it couldn’t find any new data in the pipeline then it sleeps otherwise it ingests that data in our case we are saving quotes to a text file.

```
import time

from pipeline.redis\_client import RedisClient

class QuotesConsumer:  
    redis\_client = RedisClient()  
    sleep\_seconds = 1

    def run(self):  
        while True:  
            if self.redis\_client.get\_items\_in\_pipeline() == 0:  
                print(f'No new data in pipeline, sleeping for {self.sleep\_seconds} seconds...')  
                time.sleep(self.sleep\_seconds)  
                self.sleep\_seconds += 1  
                continue

            self.sleep\_seconds = 1  
            data = self.redis\_client.get\_data\_from\_pipeline()  
            print(f'Obtained data from pipeline, saving to file...')  
            with open('quotes.txt', 'a+') as file:  
                file.write(data.get('quote'))

if __name__ == "__main__":  
    consumer = QuotesConsumer()  
    consumer.run()
```


After this step, we can run scrapy spider and consumer independently which helps us in streaming data at a very high speed as data production and consumption are independent of each other.



