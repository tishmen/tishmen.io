---
title: "Scrapy With Redis - Dockerized Approach"
date: 2018-03-26T01:25:08+02:00
---

I've been checking out couple of open source projects lately for running Scrapy spiders in a distributed way. There are some really good projects out there like [Frontera](https://github.com/scrapinghub/frontera) and [Scrapy-Cluster](https://github.com/istresearch/scrapy-cluster). While I'm pretty sure they are doing a great job at distributed crawling, I wanted to build something simple that I can use for quickly hooking up my old spider codes. I'm a huge fan of Docker and microservices, so I decided to build something on top of it. Here is what I came up with.

## Installing Docker

I'm using Ubuntu 16.04 as my primary OS. We are going to start off with installing Docker and adding it to the docker user group.

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt-get update && sudo apt-get install -y docker-ce
$ sudo usermod -aG docker ${USER}
```

After all of this is done you can check that Docker is working by using the following command.

```bash
$ docker -v
Docker version 18.03.0-ce, build 0520e24
```

## Running Redis and Webdis

We are going to have 3 services in our project managed by Compose. One of them is Redis based crawler. The other 2 services are Redis itself and [Webdis](https://github.com/nicolasff/webdis) - an HTTP API for Redis.

We will be scheduling urls to crawl in a Redis queue by despatching HTTP requests to Webdis. We can use this setup for starting new crawls from withing spiders and item post-processing as well.

Let's start by defining docker-compose.yml file with the following content, and bringing up Compose with the up command.

```yaml
version: '3.3'

services:
  redis:
    image: redis
    ports:
    - 6379:6379
    networks:
      - net

  webdis:
    image: 'anapsix/webdis'
    ports:
    - 7379:7379
    depends_on:
      - redis
    networks:
      - net

networks:
  net:
    driver: bridge
```

```bash
$ docker-compose up --build
```

Docker Compose will output all of it services logs to the console, and we can monitor them in real time.

```
webdis_1  | writing config..
redis_1   | 1:C 26 Mar 09:20:56.800 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1   | 1:C 26 Mar 09:20:56.800 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=1, just started
...
```

We should now have a running Webdis container connected with Redis. Let's use curl to check it out.

```bash
$ curl http://localhost:7379/SET/hello/world
{"SET":[true,"OK"]}
$ curl http://127.0.0.1:7379/GET/hello
{"GET":"world"}
```

Nice. We can set and retrieve keys and values in Redis. Now we can continue with creating the Scrapy project structure and developing our first crawler.

## Scrapy project structure

I use somewhat different structure for my Scrapy projects, then the one you get by running scrapy startproject. Here is how it looks like:

```
.
├── quotesbot
│   ├── __init__.py
│   ├── items.py
│   ├── settings.py
│   └── spiders.py
└── scrapy.cfg
```

We have the quotesbot folder where all of the spider files are found. There is also scrapy.cfg configuration file.

```
[settings]
default = quotesbot.settings
```

We point to the location of scrapy settings file in scrapy.cfg. We will be adding Scrapy and Scrapy-Redis settings in that file. The settings file should end up looking something like this. Feel free to change BOT_NAME and SPIDER_MODULES to point to your spider.

```python
BOT_NAME = 'quotesbot'
SPIDER_MODULES = ['quotesbot.spiders']
REDIS_HOST = 'redis'
SCHEDULER = 'scrapy_redis.scheduler.Scheduler'
DUPEFILTER_CLASS = 'scrapy_redis.dupefilter.RFPDupeFilter'
ITEM_PIPELINES = {
    'scrapy_redis.pipelines.RedisPipeline': 300
}
```

## Scrapy spider code

Let's use [quotesbot](https://github.com/scrapy/quotesbot) as our starting point. Quotesbot is a project developed by [Scrapinghub](https://scrapinghub.com/) in which they show off Scrapy capabilities. It crawls http://quotes.toscrape.com for popular quotes, authors and tags.

First thing we need to do is define the items which will be exported by our crawl.

```python
from scrapy import Item, Field

class QuoteItem(Item):
    text = Field()
    author = Field()
    tags = Field()
```

Then we continue with developing the Scrapy spider code which is going to be quite short:

```python
from scrapy import Request
from scrapy_redis.spiders import RedisSpider
from quotesbot.items import QuoteItem

class QuotesSpider(RedisSpider):
    name = 'quotes'

    def parse(self, response):
        for quote in response.xpath('//div[@class="quote"]'):
            item = QuoteItem()
            item['text'] = quote.xpath('./span[@class="text"]/text()').extract_first()
            item['author'] = quote.xpath('.//small[@class="author"]/text()').extract_first()
            item['tags'] = quote.xpath('.//div[@class="tags"]/a[@class="tag"]/text()').extract()
            yield item
        next_url = response.xpath('//li[@class="next"]/a/@href').extract_first()
        if next_url:
            yield Request(response.urljoin(next_url))
```

As you can see there is one major difference from Scrapinghub own quotesbot project. We are using RedisSpider as a spider base, instead of the default Scrapy spider. Redis spider reads urls from Redis and processes them one after another. If the spider yields more Scrapy requests, those will be processed before fetching another url from redis.

## Tying everything together

In the last step we are going to add the spider as a Compose service. Here is the service definition:

```yaml
scrapy:
  build:
    context: ./scrapy
  command: scrapy crawl quotes
  depends_on:
    - redis
  networks:
    - net
```

Once all of that is done we can start our Compose services:

```bash
$ docker-compose up --build
```

Check the services logs to make sure there are no errors while starting off. Once everything is up and running, we can drop down to console and use curl for scheduling urls. We are going to use Webdis as the API and the call will be the following:

```bash
$ curl "http://127.0.0.1:7379/LPUSH/quotes:start_urls/http%3A%2F%2Fquotes.toscrape.com"
```

In the other terminal window where Compose is running you should see Scrapy logs coming in.

```bash
scrapy_1  | 2018-03-25 22:58:59 [scrapy.utils.log] INFO: Scrapy 1.5.0 started (bot: quotesbot)
scrapy_1  | 2018-03-25 22:58:59 [scrapy.utils.log] INFO: Versions: lxml 4.2.1.0, libxml2 2.9.8, cssselect 1.0.3, parsel 1.4.0, w3lib 1.19.0, Twisted 17.9.0, Python 3.6.4 (default, Mar 14 2018, 17:49:05) - [GCC 4.9.2], pyOpenSSL 17.5.0 (OpenSSL 1.1.0g  2 Nov 2017), cryptography 2.2.1, Platform Linux-4.13.0-37-generic-x86_64-with-debian-8.10
...
```

Cool. We managed to run our first crawl. Now we need to decide what are we going to be doing with the items, consider post-processing and database storage. I'm thinking of couple of options here, depending on whether I go the SQL or NoSQL route. In the perfect world we would have some way of visualizing the crawled data, but I will leave that for another post.

All of the code for this post can be found in my GitHub repository: https://github.com/tishmen/quotesbot-redis. Feel free to pull, fork or star it. If you have any questions you can comment on Github Issues: https://github.com/tishmen/quotesbot-redis/issues.
