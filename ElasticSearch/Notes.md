# Complete Guide to Elasticsearch

## Introduction
**Elasticsearch** is an open source analytics and full-text search engine. It is easy to use and highly scalable.
It is written in java and built on top of **Apache Lucene**. 

In elasticsearch data is stored as _documents_, as rows in a relational database. A document's data is separated into _fields_, as columns in a table. A document is a simple json

```json
{
    "firstName": "foo",
    "lastName": "bar",
    "interests": ["golf", "tennis"]
}
```

### Querying Elasticsearch
The way we interact with elasticsearch is through rest apis.

### Elastic Stack
Elasticsearch is the core component of the **elastic stack**.

- **Kibana**: analytics and visualization platform
- **Logstash**: born to be used to process application logs (through **filter plugins**) and send to elasticsearch, now is a data processing pipeline, to handle events (sent from **input plugins**) and forward them on multiple destinations (called **output plugins**, or **stashes**)

```
input {
    file {
        path => "path/to/apache_access.log"
    }
}

filter {
    if [request] in ["/robots.txt", "favicon.icon"] {
        drop {}
    }
}

output {
    file {
        path => "%{type}_%{+yyyy_MM_dd}.log"
    }
}
```

- **x-pack**: add additional features to the elasticsearch and kibana, like security providing authentication and authorization or monitoring providing details about hardware metrics. Moreover, with **x-pack** we can deal with
    - _monitoring_
    - _alerting_
    - _reporting_
    - _machine learning_: for forecasting
    - _graph_ analyze the relationships in data
    - _elasticsearch SQL_: query with SQL
- **beats**: collection of data shippers that send data to elasticsearch or logstash


The **ELK** stash is the acronym for Elasticsearch, Logstash and Kibana.

To summerize:
- Data ingestion: `beats`, `logstash`
- Search, analyze, store data: `elasticsearch`
- Data visualization: `kibana`


