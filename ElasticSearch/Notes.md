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
    "_source": {
        "name": "Andersen",
        "country": "Denmark"
    }
}
```

Every document is stored in an `index`. Documents belonging to an index should be logically related.

### Inspecting the Cluster
To inspect the cluster we use the rest apis. The most common verbs used are `GET`, `PUT`, `POST` and `DELETE`. 
Using the dev tools, the path of the http requests are prepended with the elastic search cluster's network address (specified in the configuration file). 

An example request is

```
GET /_cluster/health
GET /_cat/nodes?v  # v stands for verbose
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
In elasticseach, sharding is the mechanism to divide indexes into smaller pieces, that can be put on one or multiple machines. Each piece is called `shard`, and the main purpose is to horizontally scale the data volume. 

A `shard` is technically an **Apache Lucene Index**, and it can be thoughy as an independent index. Moreover, an elasticsearch index consists in one or more Apache Lucene indeces. A shard has no predefined size; it grows as documents are added to it and may store up to about two billion documents. With sharding we can improve a lot the performance, because of the parallelization of queries on multiple nodes increase the throughput of an index. 

By default, now an index contains a single shard but up to version 7.0.0 an index was created with 5 shards, leading to _over-sharding_. We can increase the number of shards with `split` api or reduce it with `shrink` api.

### Replication
In elasticsearch, replication is supported natively and enabled by default. Replication is configured at the index level and it works by creating copies of shards, referred to as `replica shards`. A shard that has been replicated is called `primary shard`. <u>A primary shard and its replica shards are referred to as a `replication group` </u>. 

Replica shards are a complete copy of a shard, and a replica shard can serve search requests, exactly like its primary shard. The number of replicas can be configured at index creation. Of course, replicas are stored on a different nodes in respect of the primary shard to handle resilience.

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
```

| health | status | index                | pri   | rep   | ... |
| :----: | :---:  | :------------------: | :---: | :---: | --- |
| yellow | open   | pages                | 1     | 1     | ... |
| green  | open   | .kibana_task_manager | 1     | 0     | ... |
| green  | open   | .kibana_1            | 1     | 0     | ... |

If we have a cluster with only a node the status of the index will remain `yellow`, because the replica it is defined but not created, because a replica and its shard are always allocated on different nodes. Note also that replicas set for the kibana indeces are configured with one shard and zero replicas. However, if we add another node on the cluster, the `rep` will increase of one. This is because the kibana indeces are configured with a parameter called `auto_expand_replicas`, with a value from `0` to `1` that allows to dynamically changing the number of replicas depending on how many nodes the cluster has. 

To confirm that, we can inspect the shards 

```
GET /_cat/shards?v
```

| index                 | shard  | prirep   | state      | node    | ... |
| :-------------------: | :---:  | :------: | :--------: | :-----: | --- |
| pages                 |    0   |    p     | STARTED    | _node_  | ... |
| pages                 |    0   |    r     | UNASSIGNED | _node_  | ... |
| .kibana_task_manager  |    0   |    p     | STARTED    | _node_  | ... |
| .kibana_1             |    0   |    p     | STARTED    | _node_  | ... |

we see there are two shards because one is the replica, indicated with `r`, and one is the primary shard, `p`. 

### Snapshot
Elasticsearch supports taking snapshots as backups and they can be used to restore to a given point in time. Snapshots can be taken at the index level or for the entire cluster. They use snapshots for backups, and replication for high availability and performance. 


## Managing Documents


### Create and delete and index

To delete an index called "pages"

```
DELETE /pages
```

To create an index "products" with custom settings

```
PUT /products
{
    "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 2
    }
}
```

### Create a documment
To create a document (just needs to be a valid json object) and add it to an index

```
POST /products/_doc
{
    "name": "Coffee Maker",
    "price": 65
}
```

If we want to create a doc with _id to 100
```
PUT /products/_doc/100
{
    "name": "Coffee Maker",
    "price": 65
}
```

To retrieve a document
```
GET /products/_doc/<id>
```

The document retrieved with the `GET` request is in the response under the `_source` field.

We can change the value `action.auto_create_index` if we want to create the index automatically if it does not exist or if we want to make elasticsearch throwing an error.

To update a field of a document

```json
POST /products/_update/100
{
    "doc": {
        "price": 66
    }
}
```

To add a new field 
```json
POST /products/_doc/100
{
    "doc": {
        "tags": ["electronics"]
    }
}
```

### Documents Immutability
Documents are **immutable**: when updating them, elasticsearch simply replace the document with a new one. In particular, elasticsearch performs the following steps:

- retrieve the current document
- changes the fields values
- replace the previous document with the new one

we could do this by ourselves, but we would have added network traffic. 

### Scripted Updates
Elasticsearch allows to change documents values by referencing them (e.g. increase a counter by two). We use the `update` api, specifying a script object in the request body.

Decrease the price field by one of the document having id 100
```
POST /products/_update/<doc_id>
{
    "script": {
        "source": "ctx._source.price--"
    }
}
```

To set the price to 10
```
POST /products/_update/<doc_id>
{
    "script": {
        "source": "ctx._source.price = 10"
    }
}
```

We can also set parameters to be used in the script by specifying the `params` field in the `script` object, that is a map of key values pairs
```
POST /products/_update/<doc_id>
{
    "script": {
        "params": {
            "quantity": 4
        }
        "source": "ctx._source.price -= params.quantity"
    }
}
```

where `ctx` is a variable that stands for *context*. 

When running an update, in the response there is the field `result` that is `updated` if the document was updated successfully. A possible value however is `noop` (a.k.a. *no operation*), that happens for example when we try to update a field value with the existing value. This is however not the case with scripted updates, where `result` will always be `updated`, even if no field values actually changes.
We can change this behavior if we set the operation within the script. For example, we can write a script that modify a document based on a condition

```
# set price to 10 only if the price is not 20
POST /products/_update/100
{
    "script": {
        "source": """
            if (ctx._source.price == 20) {
                ctx.op = 'noop';
            }
            ctx._source.price = 10;
        """
    }
}
```

to run a complicated script, just use triple quotes `"""`. The response will contain `result` to `noop`. 

If we run the following script instead, we always get in the response `result` to `updated`

```
POST /products/_update/100
{
    "script": {
        "source": """
        if (ctx._source.in_stock > 0) {
            ctx._source.in_stock--;
        }
        """
    }
}
```

Altough not used so much, we can delete a document setting `ctx.op` to `delete`. In the response will be `result` to `deleted`

```
POST /products/_update/100
{
    "script": {
        "source": """
        if (ctx._source.in_stock <= 1) {
            ctx.op == "delete"
        }
        ctx._source.in_stock--;
    }
}
```

### Routing
To improve performance, elasticsearch knows in which shard a document is stored using the formula

```
shard_numnber = hash(_routing) % number_of_primary_shards
```

where `_routing` is by default the `_id`, but another strategy can be used. In the second case, when retrieving a document, elasticsearch returns the `_routing` document value, otherwise the parameter is omitted.

Because of the formula, when changing the number of shards, elasticsearch needs to re-index all documents and replace the index with a new one, otherwise it will be unable to find the document given an `_id` because it could happen that it search the document in the wrong shard. 

The routing strategy helps to equally distribute documents across shards. 


### How Elasticsearch reads and writes data
When a get document by id request is made, the node that handles the request is called `coordinating node`. By using the routing strategy, the coordinating node will find the replication group (i.e. the primary shard and its replicas) and to choose to which shard run the query elasticsearch uses a technique called `Adaptive Replica Selection` - `ARS`. This technique allows to understand which is the best shard to run the query. 

When a write request is made, again the node that handles the request is called the `coordinating node`. The coordinating node identifies the replication group holding the document and it does not update on every shard at the same time, but instead it performs the update on the primary shard, that is responsible to validate the request. The primary shard then performs the write operation locally, before forwarding it to the replica shards in parallel. Note that the operation will succeed even if the operation cannot be replicated to the replica shards. 

Let's assume we have a replication group with a primary shard and two replicas. It could happen that the update request reaches one replica but not the other one because of an hardware failure on the node holding the primary shard. This bring to a disalignment across shards. To handle such scenarios, elasticsearch uses two parameters: **primary term** and **sequence number**, or `_primary_term` and `_seq_no`. 

Primary terms are a way for elasticsearch to distinguish between old and new primary shards by increase `_primary_term` when the primary shard of a replication group has changed. Hence, primary term is essentially a count for how many times the primary shard has changed. The primary term is appended to write operations. Therefore, in the example the primary term for the replication group would be increased by on because the primary shard failed and one of the replica shards was promoted to be the next primary shard. The primary terms for all replication groups are persisted in the cluster's state. This enables the replica shards to tell whether or not the primary shard has changed since the operation was forwarded.

The sequence number is essentially just a counter that is incremented for each operation, at least until the primary shard changes. The primary shard is responsible for increasing this number when it processes a write request. In this way, sequence numbers enable elasticsearch to know in which order operations happened on a given primary shard. 

Together, primary terms and sequence numbers allow elasticsearch to recover from a primary shard failure (e.g. network error) because they enable elasticsearch to more efficiently figure out which write operations need to be applied. However, for large indices, this process is really expensive: to speed things up, elasticsearch uses **checkpoints**. 

Checkpoints can be global or local, and are essentially sequence numbers. Each replication group has a global checkpoint, instead each replica shard has a local checkpoint. Global checkpoints is the sequence number that all active shards within a replication group have been aligned at least up to. That is, any operation containing a sequence number lower that the global checkpoint have already been performed on all shards within the replication group. If a primary shard fails and rejoins the cluster at a later point, elasticsearch only needs to compare the operations that are above the global checkpoint that is last knew about. Likewise, if a replica shard fails, only the operations that have a sequence number higher than its local checkpoint need to be applied when it comes back. This essentially means that to recover, elasticsearch only needs to compare the operations that happened when the shard is gone, instead of the entire history of the replication group.  


### Document Versioning
Elasticsearch supports versioning with the `_version` metadata field associated to each document. However, this type of versioning does not allow to retrieve an old version of a document, but only allows to keep track of how many times a document has been changed. The value is an integer and starts at 1. It is incremented by one when modifying a document. When deleting a document, the version number is retained for 60 seconds: if in this interval of time another document with the same id is created, then the document will have the same document id incremented by one. This behavior can be configured changing the `index.gc_deletes` setting. 

This type of versioning, that is the default one, is called **internal versioning**. Elasticsearch supports also the **external versioning**, that is useful when versions are maintained outside of elasticsearch such as within a database. For example, when we use a relational database as the primary data store and index data into elasticsearch to make it searchable. To use external versioning, we specify both the elasticsearch version as well as the version type.

```
PUT /products/_doc/123?version=521&version_type=external
{
    "name": "coffee maker",
    "price": 64,
    "in_stock": 10
}
```

The point of this type of versioning is just to tell how many times a document has been modified. It was used to do optimistic concurrency control, but better mechanisms are used now and this approach is now obsolete. 


### Optimistic Concurrency Control
Optimistic concurrency control prevent overwriting documents inadvertently due to concurrent operations.

In the old way we used to use the `_version` parameter, in the sense that the requests contained the `_version` of the document. If two requests sent an update request, the second would have failed because the `_version` wouldn't match (because the first had updated the `_version` of the document). 

In the new approach, instead of sending along the request the `_version` parameter, `_primary_term` and `_seq_no` are included. In particular, we add the `if_primary_term` and `if_seq_no` in the query parameters.

```
POST /products/_update/100?if_primary_term=1&if_seq_no=71
```

 What to do in the case in which the update fails is demanded to the application logic which called elasticsearch.


### Update by query
To update by query we use the `_update_by_query` operation. In the following example, we run the script on every document 

```
POST /products/_update_by_query
{
    "script": {
        "source": "ctx._source.in_stock--"
    },
    "query": {
        "match_all": {}
    }
```

The query creates a snapshot to do optimistic concurrency control. Once the snapshot is taken, a search query is sent to each of the index's shards, in order to find all of the documents that match the supplied query. Whenever a search query matches any documents, a bulk request is sent to update those documents. A bulk operation basically allows to update multiple documents with a single request. Th `batches` key with the results specifies how many batches were used to retrieve the documents.The query uses the `scroll` api internally, which is a way to scroll through result sets. The point of doing this is to be able to handle queries that may match thousand of documents. 

Each pair of search and bulk requests are sent sequentially, that is one at a time. If an error occures while performing either the search query or the bulk query, elasticsearch will retry up to ten times. The number of retries is specified within the `retries` key, for both the search and the bulk queries. If the query is still not successful, the whole query is aborted. The failures will then be specified in the results, within the `failures` key. **Even if the query is aborted, there is no roll back operation**: so even if the update query is aborted and some documents have been updated, they are not rolled back. 

The reason why elasticsearch takes a snapshot of the index is to ensure that the updates are performed on the basis of the current state of the index. For an index where documents are indexed, modified and deleted frequently it is not unlikely that something has changed from when elasticsearch received the query, to when it finishes processing it. 

If a document has been modified since taking the snapshot, the query is aborted: this is checked with the document's primary term and sequence number. To count version confilicts instead of aborting the query, the `conflicts` option can be set to `proceed`. The `conflict` parameter can be set inside the body of the request or as a query parameter. 


### Delete by query
The delete by query api is similar to the update query api. In the following example, the query deletes all documents in the index `products`. Delete by query works in the same exact way as the update by query: we can set `conflicts` to `proceed` if we want to ignore conflicts. 

```
POST /products/_delete_by_query
{
    "query": {
        "match_all": {}
    }
```

### Batch Processing
With `bulk` apis we can index, update or delete multiple document at a time. We can do this processing individual requests in batches, specifically by using the bulk api. The **bulk** api expects data formatted using the NDJSON specification. This API accepts a number of lines, each separated by a newline character, being either `\n` or `\r\n`

In the request we can specify one of the following requests: `index`, `create`, `update` or `delete`. 

In the following request the `_id` is optional, if not specified elasticsearch will create one. Although similar, `index` and `create` operations differs because the first if the document exist it replace it, the second would fail instead. 

```
POST /_bulk
{ "index": { "_index":  "products", "_id": 200} }           # define the operation
{ "name": "espresso machine", "price": 199, "in_stock": 5 } # define the document
{ "create": { "_index":  "products", "_id": 201} }          # define the operation
{ "name": "milk frother", "price": 149, "in_stock": 15 }    # define the document
```

In the `items` response field for each of action we find the corresponding result.

To update with `bulk` api (we can add a complicated script)

```
POST /_bulk
{ "update": { "_index": "products", "_id": 201 } }
{ "doc": { "price": 129 } }
{ "delete": { "_index": "products", "_id": 200 } }
```

To avoid typing everytime the index in which we want to perform the operations we can specify it in the path
```
POST /products/_bulk
{ "update": { "_id": 201 } }
{ "doc": { "price": 129 } }
{ "delete": { "_id": 200 } }
```

Note that the http `Content-Type` header should be set as `Content-Type: application/x-ndjson`, even if `Content-Type: application/json` would be accepted. 
Moreover, each line must end with a newline character `\n` or `\r\n`: this includes the last line. Hence, in a text editor the last line should be empty. 
Finally, a failed action will not affect other actions: neither will the bulk request as a whole be aborted. This is one of the reason why the bulk api returns detailed information about each action. The order of the entries in the `items` key is the same in which the actions were specified in the request. 

The bulk api supports optimistic concurrency control: we can include the `if_primary_term` and `if_seq_no` parameters within the action metadata

### Importing data with cURL
If the bulk request body is very long, it is convenient to write it down to a file (e.g. `products-bulk.json`) and read from it using a tool like `cURL`. The request would be similar to the following

```
curl -H "Content-Type: application/x-ndjson" -XPOST http://localhost:9200/products/_bulk --data-binary "@products-bulk.json"
```















