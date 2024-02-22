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

## Fundamentals

### Basic architecture
Data are stored in one or more nodes which compose a cluster. Nodes can be on the same physical or virtual machines, or docker container. We can have more than one cluster, but they are completely isolated by default. The number of nodes of a cluster can decrease or increase over time. Data in elasticsearch are called `documents`

Documents are just json files, that will be stored along with some metadata

```json
{
    "name": "Andersen",
    "country": "Denmark"
}

{
    "_index": "people",
    "_type" : "doc",
    "_id"   : "123",
    "_version": 1,
    "_seq_no": 0,
    "_primary_term": 1,
    "_source": "{
        "name": "Andersen",
        "country": "Denmark"
    }
}
```

Every document is stored in an `index`. Documents belonging to an index should share common semantic characteristics.

### Inspecting the Cluster
To inspect the cluster we use the rest apis. The most common verbs used are `GET`, `PUT`, `POST` and `DELETE`. 
Using the dev tools, the path of the http requests are prepended with the elastic search cluster's network address (specified in the configuration file). 

An example request is

```
GET /_cluster/health
GET /_cat/nodes?v
GET /_cat/indices?v&expand_wildcards=all
```

where `_cluster` specifies the API we'll call and `health` the command to execute. The leading slash is optional, if omitted elasticsearch will add it for us. Indexes starting with `.` are system indexes.

All the queries we can run them with postman or curl

```
> curl        http://localhost:9200
> curl -X GET http://localhost:9200
> curl -X GET --insecure http://localhost:9200
> curl --cacert config/certs/http_ca.crt -u elastic -X GET http://localhost:9200
> curl --cacert config/cert/http_ca.crt -u elastic:<elastic_user_pwd> -X GET -H "Content-Type:application/json" https://localhost:9200/products/_search -d "{ \"query\": { \"match_all\": {} } }"
```

### Sharding and scalability
In elasticseach, sharding is the mechanism to divide indicies into smaller pieces, that can be put on one or multiple machines. Each piece is called `shard`, and the main purpose is to horizontally scale the data volume. 

Although it is an approximation, a `shard` is an independent index and it is in particular an Apache Lucene `index`. Moreover, an elasticsearch index consists in one or more Apache Lucene indeces. A shard has no predefined size; it grows as documents are added to it and may store up to about two billion documents. With sharding we can improve a lot the performance, because of the parallelization of queries on multiple nodes increase the throughput of an index. 

By default, now an index contains a single shard but up to version 7.0.0 an index was created with 5 shards, leading to _over-sharding_. We can increase the number of shards with `split` api or reduce it with `shrink` api.

### Replication
In elasticsearch, replication is supported natively and enabled by default. Replication is configured at the index level and it works by creating copies of shards, referred to as `replica shards`. A shard that has been replicated is called a `primary shard`. A primary shard and its replica shards are referred to as a `replication group`. 

Replica shards are a complete copy of a shard, and a replica shard can serve search requests, exactly like its primary shard. The number of replicas can be configured at index creation. Of course, replicas are stored on a different nodes in respect of the primary shard. 

We can increase a lot the query throughput with replication: replica shards of a replication group can serve different search requests simultaneously, increasing the number of requests that can be handled at the same time. Elasticsearch intelligently routes requests the best shard, and cpu parallelization improves performance if multiple replica shards are stored on the same node. 

To create an index called `pages`, just run `PUT /pages`. Since no other options are specified, the default will be applied. Hence, a primary shard and a replica shard will be created. If we inspect the cluster we'll see that its status is changed from `green` to `yellow`

```
GET /_cluster/health

{
    "cluster_name": "elasticsearch",
    "status"      : "yellow",
    ...
}
```

If we run the `_cat` operation with `v` for verbose

```
GET /_cat/indices?v

| health | status | index                | pri   | rep   | ... |
| :----: | :---:  | :------------------: | :---: | :---: | --- |
| green  | open   | .kibana_task_manager | 1     | 0     | ... |
| yellow | open   | pages                | 1     | 1     | ... |
| green  | open   | .kibana_1            | 1     | 0     | ... |
```

If we have a cluster with only a node the status of the index will remain `yellow`, because the replica it is defined but not created, because a replica and its shard are always allocated on different nodes. Note also that replicas set for the kibana indeces are configured with one shard and zero replicas. However, if we add another node on the cluster, the `rep` will increase of one. This is because the kibana indeces are configured with a parameter called `auto_expand_replicas`, with a value from 0 to 1 that allow to dynamically changes the number of replicas depending on how many nodes the cluster has. 

To confirm that, we can inspect the shards 

```
GET /_cat/shards?v

| index                 | shard  | prirep   | state      | node    | ... |
| :-------------------: | :---:  | :------: | :--------: | :-----: | --- |
| pages                 |    0   |    p     | STARTED    | <node>  | ... |
| pages                 |    0   |    r     | UNASSIGNED | <node>  | ... |
| .kibana_task_manager  |    0   |    p     | STARTED    | <node>  | ... |
| .kibana_1             |    0   |    p     | STARTED    | <node>  | ... |
```

we see there are two shards because one is the replica, indicated with `r`, and one is the primary shard, `p`. 

### Snapshot
Elasticsearch supports taking snapshots as backups and they can be used to restore to a given point in time. Snapshots can be taken at thge index level or for the entire cluster. They use snapshots for backups, and replication for high availability and performance. 








