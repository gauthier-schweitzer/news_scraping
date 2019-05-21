# Extracting information about clients in newspapers

## Summary
**Objective : ** Provide bank advisors with useful information about their clients using NLP

**Method : **
Scrape all newspapers in one of the French regions and identify clients in the articles database using a powerful text-engine. Instead of setting up a static analysis, we've prefered enabling dynamic searches to better understand the usecase.

**Stack : **
- Scraping performed using Scrapy
- Indexing and research performed using Elasticsearch, hosted on Elastic Cloud
- Analysis conducted in Python, by querying the cluster using `pycurl`

**Remark : **
This folder is dedicated to the creation of a static database. Another one `not_news_scraping` serves another purpose! each day and automatically :
- scraping of the most recent news (from the previous date)
- indexing in ES
- loop to identify news related to clients


## Details of the pipeline
1. Scraping of newspapers using a Web Crawler built on Scrapy for Python.
  Three newspapers have their crawler ready : la depeche (600 000 articles collected), midi libre (700 000) and Ouest France (0 article collected since there exists a paywall that could be bypassed should we subscribe)
2. Preprocess the database in Pyhon to add the extra columns that we need
3. Adapt the json to create an [Elasticsearch-compliant json](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) (it is necessary to specifiy the index for each article)
4. Write the bulk query in Python and copy-paste it in the terminal to index all of the articles
5. Query performed using a Python notebook, that features some interactive apps to enable the user to interact with the database

## Repo structure
- ladepeche : crawler and Python preprocessing file. The `base` directory contains all of the necessary json
- midilibre : similar
- ouestfrance : similar, but wihout any data collected due to the paywall
- elastic : contains the notebook used to analyze the Articles

## Zoom on the Scraping
I have followed most of the Udemy course [Powerful Web Scraping & Crawling with Python](https://www.udemy.com/scrapy-tutorial-web-scraping-with-python/) before creating my first scraper in Scrapy. Then, I have decided to choose a crawler, instead of a scraper. The choice has been made due to the structure of the website to be crawled. URLs are based on the title of the article: thus, it is impossible to loop on them as we would do in a forum, for instance. What we can do instead is use a crawler, starting from a given article. Then, it scrapes the page and stores all of the URLs it finds on the page. After that, it follows one of the URLs and keeps on doing the same thing. To avoid issues, you can define domains that can be followed and deny some that should not (social networks for instance). You can also specify a max depth for the URLs to be followed.
Scrapy includes several tools to avoid being banned and to scrape websites efficiently (user-agent, middlewares, ...)

Before building such crawlers, that can become fairly complex, I suggest you first ramp up by exploring the scrapy shell that offers an IDE to use scrapy tools.

## Zoom on Elasticsearch part

### Which elastic ?
I first tried to use Elasticsearch setup on my own laptop before moving to Elasticloud for convenience, speed and ease of deployment. There exists a 14 days trial that enables you to discover the power of Elasticsearch and decide if the tool is the right one for you.

### How to ramp up ?
I have followed the whole Udemy course [Powerful Guide to Elasticsearch](https://www.udemy.com/elasticsearch-complete-guide/learn/v4/t/lecture/7470446?start=0) to know what was the technology about. It is absolutely possible to setup a cluster only by looking at the right parts of the documentation (very useful). However, a MOOC can be useful to better understand the infrastucutre part and how it influences the performance of the search.

### How to choose the infrastructure ?
Regarding the choice of the cluster, I have used the trial version and have been jumping from one trial to another. From a very very macro point of view, the main determinants of the price of the cluster will be :
- the type of cluster : I/O optimized, memory optimized, storage optimized
- the size of the Memory (used for computation)
- the size of the storage (not very binding in our case)
- the number of zones (used for availability and increasing performance in case queries are made from several continents)

EDIT : We have now opted for a paid version of Elastic Cloud. To limit fees, I have chosen one of the smallest possible architectures :
- Fault tolerance limited to 1 zone, meaning that we only have 1 node for the data (limits availability and speed to a smaller extente, but price is proportional to the number of nodes)
- RAM per Node : 2GB. 1 GB would have been too few do to the size of the articles and more would be overkill. I have chosen that size by trial and error

Thus, with that configuration, hourly price is slightly less than $0.05, leading to a very reasonable monthly price of $36,35.

### How to modify the infrastructure?
Using [elastic cloud dashboard](https://cloud.elastic.co/), "edit deployment", you can easily modify the structure of your elastic cluster. However, you can easily make mistakes. By default in 6.x versions of elasticsearch, each index is being created with 5 shards and 1 replica. Therefore, when you modify your infrastucture, you have to consider it.

In my situation, moving from the trial to a small cluster with only one node implied changes in my index structure. Since I now have one node only, it is impossible to have 1 replica for each of the shards. Therefore 5 shards were unassigned. If there are unassigned shards and you want to understand why, you can type in Kibana Dev Tools  `GET _cluster/allocation/explain?pretty`. The error indicated that I already had a copy of the shard on my node and that the replica shard could not be assigned to the node. It appears normal. The original shard is already located on the node, therefore the copy cannot be on the same node.
In that precise case, you have to set the number of replicas to 0 for your index. Mine is named 'article'. To perform the change, you just have to type :
`PUT article/_settings
{
  "number_of_replicas": 0
}`

After the changes, check the health of each of your shards by looking at the piechart (on cloud dashboard, within your deployment, by clicking on "Elasticsearch") or typing : `GET _cat/indices?v`. Everything should be green.

### Steps to use elasticsearch

To use elasticsearch, a few steps are required (if you use elasticloud):
1. Create an account
2. Open Kibana and use the "Dev Tools" tab to write queries
3. Create an index and specify a mapping (more or less what is known as the "schema" in structured databases)
4. Add your first documents directly in Kibana with PUT
5. To go further and add multiple documents at the same time, use the _bulk API. It is working with Windows Terminal if `curl` is installed. You can automate it in Python using the `elasticsearch` package or simply by encapsulating your curl queries in Python, using `pycurl`. What I have done is simply to automate the query writing and then simply copy-paste the queries in the terminal. This choice is not the most logical one, it was the simplest one, at a time when I could not very easily understand how the tools were working
6. Perform queries, in Kibana or in Python using the aformentioned packages. What I have chosen is to use `pycurl` to query the cluster. Once more, it would have been more efficient to use the `elasticsearch` package, but I had no time at the time to dig in this package. To better understand its use, you can refer to the `not_news_scraping` github repo that I created to index only the most recent articles scraped. To do that, when articles are scraped, they are indexed straight away, without creating a json first.

### Key moments in an elasticsearch project
In elasticsearch, the magic comes from two very strategic moments :
- the mapping time is the time of a difficult tradeoff, mainly for text field. A text field can have two forms in Elasticsearch: `text` is aimed at searches of words included in the field, while `keyword` is used for aggregations or statistics on words
  - enabling `text` and `keyword` will be more costly in terms of computation and storage
  - if you disabled one the functions, you cannot cancel it later on. You have to reindex the whole thing.
- the query time, of course. It is hard to draw general conclusions without knowing the use case. What I could suggest would be :
  - always start with simple queries (`match` or `terms` queries) that you fully understand
  - enrich them as time goes by to refine the results (`multi_match`, `bool`, `aggregations` ...). You can see that in my final code, I have several queries implemented. Since elasticsearch enables real-time analyses, I often switch queries to check the conclusions I draw on different fields. One of the strenghts of elasticsearch is the ability to perform aggregations and queries at the same time. What it means is that you can run any query to select the most relevant documents and at the same time collect some statistics about the results : count some stuffs, draw an histogram of dates ...
