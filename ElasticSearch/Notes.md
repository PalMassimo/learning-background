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

| health | status |        index         | pri | rep | ... |
| :----: | :----: | :------------------: | :-: | :-: | --- |
| yellow |  open  |        pages         |  1  |  1  | ... |
| green  |  open  | .kibana_task_manager |  1  |  0  | ... |
| green  |  open  |      .kibana_1       |  1  |  0  | ... |

If we have a cluster with only a node the status of the index will remain `yellow`, because the replica it is defined but not created, because a replica and its shard are always allocated on different nodes. Note also that replicas set for the kibana indeces are configured with one shard and zero replicas. However, if we add another node on the cluster, the `rep` will increase of one. This is because the kibana indeces are configured with a parameter called `auto_expand_replicas`, with a value from `0` to `1` that allows to dynamically changing the number of replicas depending on how many nodes the cluster has.

To confirm that, we can inspect the shards

```
GET /_cat/shards?v
```

|        index         | shard | prirep |   state    |  node  | ... |
| :------------------: | :---: | :----: | :--------: | :----: | --- |
|        pages         |   0   |   p    |  STARTED   | _node_ | ... |
|        pages         |   0   |   r    | UNASSIGNED | _node_ | ... |
| .kibana_task_manager |   0   |   p    |  STARTED   | _node_ | ... |
|      .kibana_1       |   0   |   p    |  STARTED   | _node_ | ... |

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

If we want to create a doc with \_id to 100

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

where `ctx` is a variable that stands for _context_.

When running an update, in the response there is the field `result` that is `updated` if the document was updated successfully. A possible value however is `noop` (a.k.a. _no operation_), that happens for example when we try to update a field value with the existing value. This is however not the case with scripted updates, where `result` will always be `updated`, even if no field values actually changes.
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

### Mapping and Analysis

### Analysis

Also known in this context as _text analysis_, is applicable only to text fields/values, which are analyzed when indexing documents. Therefore, everytime a document is created it goes through an analysis process. The objective is to store in data structures that are efficient for searching. When elasticsearch performs search queries, it does not use the field values stored in `_source` field.

When a text value is indexed, a so called `analyzer` is used to process the text, i.e. the value is analyzed. An analyzer consist of three building blocks: character filters, a tokenizer and token filters. The result of analyzing text values is then stored inn a searchable data structure.

**Character Filters** add, remove or change characters. An analyzer can have zero or more character filters, and they are applied in the same order in which they are specified.

An analyzer contains one **tokenizer**, which is used to split strings in token. Caracters may be stripped as part of the tokenization. For example: `"I REALLY like beer!" -> ["I", "REALLY", "like", "beer"]`.

**Token filters** receive the output of the tokenizer as input (i.e. the tokens). A token filter can add, remove or modify tokens. As with character filters, an analyzer may contain zero or more token filters, and they are applied in the same order in which they are specified. One example of token filter is the lowercase filter: `["I", "REALLY", "like", "beer"] -> ["i", "really", "like", "beer"]`.

Elasticsearch ships with a number of built-in character filters, tokenizers and token filters. It is possible to mix and match these together to form custom analyzers.

The **standard analyzer** is the one used by default and consists of zero character filters, the standard tokenizer which basically removes punctuation and split the strings in tokens based on the whitespace ` ` and the lowercase token filter which simply turn all uppercase letters in lowercase.

### Analyze API

We can use the `analyze` api to test the standard analyzer running the following query, but we can specify a different analyzer in the `analyzer` field

```
POST /_analyze
{
    "text": "2 guys walk into a bar"
    "analyzer": "standard"
}
```

We can run the test specifying the individual components of the analyzer. Let's start with the character filters: since there are not in the standard analyzer, the query must have an empty array as `char_filter` or omit the field entirely.

```
POST /_analyze
{
    "text": "2 guys walk into a bar",
    "char_filter": [],
    "tokenizer": "standard"
}
```

In this way we can specify other components to build another analyzer and test the results.

### Inverted Indices

A couple of different data structures are used to store field values. The data structure that is used for a field depends on its data type. Are used more than one data structure to ensure efficient data retrieval for different access patterns. Actually, these data structures are all handled by Apache Lucene.

One of these data structure is **inverted index**, a data structure which is basically a mapping between terms and which docuemnts contain them; where `term` is one token that is emitted by the analyzer. This mean that we have as many inverted index as the tokens. Given the two documents, we have two inverted indices

```json

{
    "name": "coffee maker",
    "description": "makes coffee super fast!",
    "price": 64
}

{
    "name": "toaster",
    "description": "makes delicious toasts",
    "price": 49
}

```

Inverted index for `name` field

|  term   | document #1 | document #2 |
| :-----: | :---------: | :---------: |
| coffee  |      x      |             |
|  maker  |      x      |             |
| toaster |             |      x      |

Inverted index for description field

|   term    | document #1 | document #2 |
| :-------: | :---------: | :---------: |
|   makes   |      x      |      x      |
|  coffee   |      x      |             |
|   super   |      x      |             |
|   fast    |      x      |             |
| delicious |             |      x      |
|   toast   |             |      x      |

What makes an inverted index so powerful is how efficient it is to look up a term and find the documents in which the term appears. Hence, if we perform a search for the term `duck`, an inverted index allows us to find in which documents such term is contained.

An inverted index is something more; it contains more information, such as data that is used for **relevance scoring**.

Moreover, inverted indexes are built for text fields only, because other data types use different data structures. For instance, `numeric`, `date` and `geospatial` fields are all stored as BKD trees, because this data structure is very efficient for geospatial and numeric range queries. Dates are included because dates are actually stored as long values internally.

### Introduction to Mapping

Mapping defines the structure of documents and how they are indexed and stored. This includes the fields of a document and their data types. As simplification, we can think of it as the equivalent of a table schema in a relational database.

In elasticsearch exist two basic approeaches to mapping: **explicit mapping** and **dynamic mapping**. With explitic mapping we defined field mappings ourselves, tipically when creating an index. With **dynamic mapping**, a field mapping will automatically be created when elasticsearch ecounters a new field: elasticsearch inspect its value to infer its data type. Explicit and dynamic mapping can be combined together.

### Data Types

Elasticsearch has many data types, the most common are

- object
- integer
- long
- text
- short
- date
- float
- boolean
- double

#### object and nested data types

The `object` data type basically covers any json object. Each indexed document may contain a field with an object as its value.
A field mapped as object has a field `properties` which is actually a mapping parameter

```json
{
  "name": "coffee maker",
  "price": 64.2,
  "in_stock": "10",
  "is_active": true,
  "manufacturer": {
    "name": "nespresso",
    "country": "switzerland"
  }
}

{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "double" },
      "in_stock": { "type": "short" },
      "is_active": { "type": "boolean" },
      "manufacturer": {
        "properties": {
          "name": { "type": "text" },
          "country": { "type": "text" }
        }
      }
    }
  }
}
```

Elasticsearch is built on top of Apache Lucene, which does not support objects; that's why elasticsearchtransforms inner objects to a format that is compatible, as part of indexing operations. Internally, objects are flattened: each level in the hierarchy is denoted with a `.`, such that there are no longer any objects, but the hierarchy is still maintained.

```json
{
  "name": "coffee maker",
  "price": 64.2,
  "in_stock": "10",
  "is_active": true,
  "manufacturer.name": "nespresso",
  "manufacturer.country": "switzerland"
}
```

Let's assume we want to index an object having an array as a field. The items of the array will be flattened, resulting in field names duplicates. The values are grouped by field name and indexed as an array

```json
{
  "name": "coffee maker",
  "reviews": [
    {
      "rating": 5.0,
      "author": "average Joe",
      "description": "haven't slept for days... amazing!"
    },
    {
      "rating": 3.5,
      "author": "John Doe",
      "description": "could be better :)"
    }
  ]
}
```

```json
{
  "name": "coffee maker",
  "reviews.rating": [5.0, 3.5],
  "reviews.description": [
    "haven't slept for days... amazing!",
    "could be better :)"
  ]
}
```

If you run a search query against one of the fields, it will search through all of the values within the array. This can be troublesome in some situations: suppose we want to run the following query

```
MATCH products WHERE reviews.author == "John Doe" AND reviews.rating >= 4.0
```

The query result will contains the product named "coffee maker", because the field values wew all mixed together when the document was indexed, so the relationship between the object keys was lost. Therefore, elasticsearch does not know that there is a relationship between "John Doe" and 3.5. Effectively, we have returned an object that has a revuew wrutte by John Doe or with a rating of at least 4: the `AND` condition was turned to an `OR` condition.

Obviously we want to not have this behavior. To solve this, there is a data type called `nested`, which is a specilized version of the `object` data type. Its purpose is to maintain the relationship between object values whan an array of objects is indexed. Using this data type enables us to query objects independently, meaning that the object values are not mixed together as you saw a moment ago. To utilize this data type, we must use a specific query, which you will see later in the course.

Let's see an example of the `nested` data type. The data type is simply set to `nested` in the same way as for other data types.
In this way, the result of the previous query would be correct, but we have to use a different syntax. In particular, we have to use the `nested query`

```json
PUT /products
{
    "mappings": {
        "properties": {
            "name": { "type": "text"},
            "reviews": { "type": "nested"}
        }
    }
}
```

Since Apache Lucene index cannot store object, inner objects are stored as documents would be. They do not show in result queries unless we query them directly. Therefore, if we index a document having ten reviews, each review will be a nested object. This would cause eleven documents to be indexed into Lucene; one for the product and one for each review.

#### keyword and text data types

The `keyword` data type should be used for fields on which we want to search for exact values. Since only exact searching is supported, this data type is used for filtering, sorting and aggregating documents. One use case is for the enum values.

On the other side, we have to use the `text` data type instead if we do not want exact matches.

### How the keyword data type works

Keyword fields are analyzed with the `keyword analyzer`, that is a `no-op` analyzer. That is, it outputs the unmodified string as a single token, which in turn is placed into the inverted index. This because this data type is used for exact matches. Moreover, this explains why it cannot be used for full text searches as `text` data types.

We can test the behvaior of the keyword analyzer through the `_analyze` api

```
POST /_analyze
{
    "text": "2 guys walk into     a bar... wow!"
    "analyzer": "keyword"
}
```

We would get the response

```json
{
    "tokens": [
        {
            "token": "2 guys walk into     a bar... wow!"
            "start_offset": 0,
            "end_offset": 53,
            "type": "word",
            "position": 0
        }
    ]
}
```

A typical use case of the keyword analyzer is with email addresses, where we would like to store them in lower case. We can achieve this customizing the keyword analyzer to use the lowercase token.

`keyword` fields are used for exact matching, aggregation and sorting

### Understanding Type Coercion

Elasticsearch checks the data types for each field value when we index a document. A part of this process is to validate the data types and reject field values that are obviously invalid.

Let's run the following queries, where the index `coercion` does not exist yet. A mapping for the “price” field has actually been created due to dynamic mapping. What Elasticsearch did, was to inspect the value that we supplied in order to figure out its data type. In this case we supplied a floating point number, so the `float` data type was used within the mapping.

As for the next query, you have probably noticed how the floating point number has been wrapped within double quotes, effectively turning it into a string value. The document was indexed correctly even though we supplied a string value instead of a floating point number, because of **type coercion**. Elasticsearch first resolves the data type that was supplied for a given field. In the case where there is a mapping for the field, the two data types are compared. Since we supplied a string for a numeric data type, Elasticsearch inspects the string value. If it contains a numeric value — and only a numeric value — Elasticsearch will convert the string to the correct data type, being “float” in this example. The result is that a string of 7.4 is converted into the floating point number 7.4 instead, and the document is indexed as if we had supplied a number in the first place. The third query won't succeed because the data type is incorrect, and Elasticsearch was unable to coerce the value into the appropriate data type.

If we retrieve the second document we indexed, the value within the source document is a string and not a float. This because the “\_source” key contains the original values that we indexed. These are not the values that Elasticsearch uses internally when searching for data. Elasticsearch searches the data structure used by Apache Lucene to store the field values. In the case of text fields, that would be an inverted index. In this example, the value has been converted into a float, even though we see a string here. Within Apache Lucene, the value is stored as a numeric value and not a string. This means that if you make use of the value within the “\_source” key, you need to be aware that the value may be either a float or a string, depending on the value that was supplied at index time.

To change the behavior, we either need to supply the correct data type in the first place, or disable coercion.

```
PUT /coercion/_doc/1
{
    "price": 7.4
}

# elasticsearch applies coercion
PUT /coercion/_doc/2
{
    "price": "20.5"
}

# an error will be thrown
PUT /coercion/_doc/3
{
    "price": "7.4m"
}
```

Coercion is not used when figuring out which data type to use for a field with dynamic mapping. This means that if you supply a string value, the data type will be set to “text” even if the string only contains a number.

In general you should always try to use the correct data types. It’s especially important when you index a new field, to ensure that the correct data type is used in the mapping — provided that you make use of dynamic mapping. Anyway, the point is that Elasticsearch does not use coercion when creating field mappings for us; it’s only used as a convenience that forgives us if we use the wrong data types.

Coercion is enabled by default to make Elasticsearch as easy and forgiving as possible.

### Understanding Arrays

In elasticsearch it does not exist the array data type because any field in elasticsearch may contain zero or more values by default: we can index an array of values without defining this within the field's mapping. Indeed, all of the following queries are valid

```
POST /_products/_doc
{
    "tags": "smartphone"
}
```

```
POST /_poroducts/_doc
{
    "tags": ["smartphone", "electronics"]
}
```

Both will end to the following mapping, that does not contains any reference

```json
{
    "products": {
        "mappings": {
            "properties": {
                "tags":
                    "type": "text"
            }
        }
    }
}
```

Internally, in case of text fields the strings are simply concatenated before being analyzed, and the resulting tokens are stored within an inverted index as normal. We can verify it running the following query

```
POST /_analyze
{
    "text": ["strings are simmply", "merged together"],
    "analyzer": "standard"
}
```

The result of this query contains tokens from both strings. The character offsters for the last two tokens, originating from the second string. The offsets don't start over from zero, but rather continue from the last offset of the previous string. The fields used to keep track of the offset of the tokens are `start_offset` and `end_offset`. The `position` field indicate the order of the token inside of the array. Hence, an array is treated as a string, obtained by concatenating all the items of the array.

In the case of non-text fields, the values are not analyzed, and multiple values are just stored within the appropriate data structure within Apache Lucene.

The array items in elasticsearch must be of the same data type. To be more precise, they can be of different data types as long as the provided types can be coerced into the data type used within the mapping. In this case, coercion must be enabled. Notice that coercion only works for fields that are already mapped. If creating a field mapping withdynamic mapping, an array must contain the same data type.

Arrays may contain nested arrays, which are flattened during indexing: `[1, [2, 3] ] -> [1, 2, 3]`. Remember to use the `nested` data type for arrays of objects if we need to query the objects independently. If we do not need to query the objects independelty, we can just use the `object` data type.

### Adding Explicit Mappings

We can add field mappings both when creating an index and afterwards. However, tipically the mapping is done right after index creation. All field mappings should be defined within a “properties” key. That’s the case for fields at every level of the hierarchy, including nested objects

```
PUT /reviews
{
    "mappings": {
        "properties": {
            "rating": { "type": "float" },
            "content": { "type": "text" },
            "product_id": { "type": "integer" },
            "author": {
                "properties": {
                    "first_name": { "type": "text" },
                    "last_name": { "type": "text" },
                    "email": { "type": "keyword" }
                }
            }
        }
    }
}
```

Now, if we try to index a document having an object in the field `author.email`, elasticsearch will throw an error.

Elasticsearch does not have a naming convention for field names, but camelCase and snake_case are the most widely used for json, so is a good idea to use one of those.

### Retrieve Mappings

To retrieve an index dynamic mapping

```
GET /reviews/_mapping
```

We can retrieve the mapping for a specific field

```
GET /reviews/_mapping/field/content
GET /reviews/_mapping/field/author.email
```

### The Dot Notation

When we added the mapping for the "reviews" index, we mapped an `author` field as an object. In Elasticsearch, objects are mapped implicitly by using the `properties` mapping parameter at each level of the hierarchy. Since the `author` field doesn’t contain any nested objects the mapping isn't that bad, but for more advanced mappings, it can look a little bit messy.

Instead of defining objects in this way, we can use an alternative approach: a **dot notation**. The way it works is that we add a dot at each level of the hierarchy, i.e. as a field separator. This format is a bit easier on the eyes, particularly if you intend to nest objects further than just one level.

```json
PUT /reviews
{
    "mappings": {
        "properties": {
            "rating": { "type": "float" },
            "content": { "type": "text" },
            "product_id": { "type": "integer" },
            "author.first_name": { "type": "text" },
            "author.last_name": { "type": "text" },
            "author.email": { "type": "keyword" }
        }
    }
}
```

If we try to retrieve the mapping, this would be exactly the same. This is because Elasticsearch translates fields using a dot notation into using the `properties` parameter behind the scenes. Therefore, we'll never see an index’ mapping contain the dot notation, as this is simply a shortcut while creating the mapping.

Note that this dot notation is not exclusive to defining field mappings; it can also be used within search queries.

### Edit mappings to existing indices

If we want to add a new field `created_at` for an existing mapping

```json
PUT /reviews/_mapping
{
    "properties": {
        "created_at": {
            "type": "date"
        }
    }
}
```

### Dates in Elasticsearch

Dates in elasticsearch can be represented in three ways:

- specially formatted strings
- a `long` number that indicates the number of milliseconds since the epoch
- a `int` number that indicates the number of seconds since the epoch

the latter is also referred as a UNIX timestamp. The epoch refers to the **1st January 1970**.

When custom format is specified, elasticsearch expects one of three formats:

- a date without time
- a date with time
- a long number representing the number of milliseconds since the epoch

In the case of string timestamps, these are assumed to be in the `UTC` timezone if none is specified. If we supply a string value, the date needs to be in the ISO 8601 format. This is the most standard date format and what is generally used besides numeric timestamps.

Independently of the way we provide the date, it will always be converted to a long number for storage. If a date includes a timezone, it will be converted to the UTC format first. All queries are then run against this long number.

Let's index a document with a date. The letter `T` is to indicate that is compatible to the ISO 8601 specification and the letter `Z` is to specify that the date is in the UTC timezone.

```
PUT /reviews/_doc/3
{
    "rating": 3.5,
    "content": "could be better",
    "product_id": 123,
    "created_at": "2015-04-15T13:07:41Z",
    "author": {
        "first_name": "Adam",
        "last_name": "Jones",
        "email": "adam.jones@example.com"
    }
}
```

We could provide the date without time. In this case, midnight is assumed.

```
PUT /reviews/_doc/3
{
    "rating": 3.5,
    "content": "could be better",
    "product_id": 123,
    "created_at": "2015-04-15",
    "author": {
        "first_name": "Adam",
        "last_name": "Jones",
        "email": "adam.jones@example.com"
    }
}
```

The date has not to be in the utc format. In this case, the utc offset can be specified instead of `Z`. In the example below, we specify that the timestamp is offset by one hour compared to UTC

```
PUT /reviews/_doc/3
{
    "rating": 3.5,
    "content": "could be better",
    "product_id": 123,
    "created_at": "2015-04-15T09:21:51+01:00",
    "author": {
        "first_name": "Adam",
        "last_name": "Jones",
        "email": "adam.jones@example.com"
    }
}
```

We can also specify the date with a long number (i.e. the number of seconds and NOT milliseconds)

```
PUT /reviews/_doc/5
{
    "rating": 4.5,
    "content": "could be better",
    "product_id": 123,
    "created_at": 1436077284000,
    "author": {
        "first_name": "Taylor",
        "last_name": "West",
        "email": "twest@example.com"
    }
}
```

Even if dates are stored in the same way, when we query them we'll see the way we put in there.

### Missing Fields

In elasticsearch all fields are optional, so we don't have to fill any of them when indexing a document. Therefore the null checks must be done on the application level. Elasticsearch will validate field values against any mapping that might exist, but it won’t reject documents that leave fields out.

When searching for documents, you just specify field names as normal, and Elasticsearch will only consider documents that contain a value for the field.

### Mapping and Analysis

When specifying mappings, apart from field’s data type we can define also some parameters available that configure the behavior of fields in various ways. Let's see some of them

#### `format` parameter

This parameter is used to specify a date format when you want to index dates that are not formatted in the default format. I recommend that you use the default date format whenever possible. In particular the ISO 8601 format is an extremely common format that should be supported by almost all systems.

We can completely customize the date format by specifying a format that is compatible with Java’s DateFormatter class. Alternatively there are a number of built-in formats available. Two of them are actually used within the default format that Elasticsearch uses. There is a rather long list of other built-in formats that we can conveniently use instead of building our own pattern.

An example could be if we wanted to specify a UNIX timestamp instead of the number of milliseconds since the epoch, in which case we could use the “epoch_seconds” format.

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "purchased_at": {
                "type": "date",
                "format": "dd/MM/yyyy"
            }
        }
    }
}
```

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "purchased_at": {
                "type": "date",
                "format": "epoch_second"
            }
        }
    }
}
```

#### `properties` parameter

We have already seen it both at the top level when defining mappings, and for nested fields. To be clear, nested fields can be for both fields of the `object` and `nested` data types.

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "sold_by": {
                "properties": {
                    "name": {"type": "text"}
                }
            }
        }
    }
}
```

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "products": {
                "type": "nested",
                "properties": {
                    "name": {"type": "text"}
                }
            }
        }
    }
}
```

Remember there is no such data type in Elasticsearch; instead, “object” fields are mapped implicitly by using the `properties` parameter instead of specifying a type.

#### `coerce` parameter

The `coerce` parameter is used to enable or disable type coercion; it’s enabled by default.

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "amount": {
                "type": "float",
                "coerce": false
            }
        }
    }
}
```

It’s also possible to configure coercion at the index level so you don’t have to specify the “coercion” parameter for every field. Since coercion is enabled by default, it only really makes sense to disable it at the index level. Fields inherit this index level setting, but you can still overwrite it if you want to.

```json
PUT /sales
{
    "settings": {
        "index.mapping.coerce": false,
    },
    "mappings": {
        "properties": {
            "amount": {
                "type": "float",
                "coerce": true
            }
        }
    }
}
```

Typically you will have fields that don’t specify the `coerce` parameter and thereby use the index level setting implicitly.

#### `doc_values` parameter
The `doc_values` parameter requires us to dive a bit deeper into Apache Lucene. We have seen what an inverted index is and how that is used to efficiently resolve which documents contain a given term. The `doc_values` has basically the same purpose.

Remember that there is no single data structure that efficiently serves all purposes. Therefore, inverted indexes are not the only data structure used for fields. 

Instead of looking up terms and finding the documents that contain them, we need to look up the document and find its terms for a field. This is what the “doc values” data structure is; pretty much the opposite of an inverted index. Besides being used for sorting and aggregations, this data structure is also used when accessing field values from within scripts.

To be clear, the “doc values” data structure is not a replacement for an inverted index, but rather an addition to it, meaning that both data structures will be maintained for a field at the same time. Elasticsearch will then query the appropriate data structure depending on the query.

We have the option of disabling it with the `doc_values` mapping parameter. The main reason for doing this would be to save disk space, because this data structure would then not be built and stored on disk.

Disable `doc_values` when we don’t need to use a field for sorting, aggregations, and scripting,

Note that if you disable doc values, you cannot change this without reindexing all of your data.

``` json
PUT /sales
{
    "mappings": {
        "properties": {
            "buyer_email": {
                "type": "keyword",
                "doc_values": false
            }
        }
    }
}
```

#### `norms` parameter

Norms refers to the storage of various normalization factors that are used to compute relevance scores. Therefore, the `norms` parameter is one of the parameter elasticsearch use to assign a document a score when a search query is performed.

The `norms` mapping parameter enable us to disable these norms. This is because they take up quite a lot of disk space, just like `doc_values` do. Not storing `norms` saves disk space, but also removes the ability to use a field for relevance scoring. That’s why we should only disable norms for fields that we won’t use for relevance scoring.

For example, we can disable `norms` on `tags` field for products because such field would almost always be used for either filtering or aggregations, and does not involve relevance scoring. 

```json
PUT /products
{
    "mappings": {
        "properties": {
            "tags": {
                "type": "text",
                "norms": false
            }
        }
    }
}
```

#### index parameter
It’s also possible to ignore a field in terms of indexing, by setting the `index` parameter to `false`. By doing so, the field values are not indexed at all, and the field therefore cannot be used within search queries. It’s important to note that the field value will still be stored within the `_source` object; it just won’t be part of the data structures used for searching.

This parameter is useful if you have a field that you don’t need to search, but you still want to store the field as part of the `source` object. Not indexing a field naturally saves disk space and slightly increases the rate at which documents can be indexed. This parameter is often used for time series data where you have numeric fields that you won’t use for filtering, but rather for aggregations. Even if you disable indexing for a field, you can still use it for aggregations. 

```json
PUT /server-metrics
{
    "mappings": {
        "properties": {
            "server_id": {
                "type": "integer",
                "index": false
            }
        }
    }
}
```

#### `null_value` parameter

If we include the field and supply `NULL` as the value, is will be ignored in Elasticsearch, meaning that it cannot be indexed or searched. The same applies to an empty array, or an array of `NULL` values.

If we want to be able to search for `NULL` values, then we can use the `null_value` parameter to replace `NULL` values with a value of your choice.

Whenever Elasticsearch encounters a `NULL` value for the field, it will index the value specified by the parameter instead, thereby making this value searchable.

There are a couple of things to note about this parameter. First, it only works for explicit `NULL` values, meaning that providing an empty array does not cause `elasticsearch` to perform the replacement. Secondly, the value that we define must be of the same data type as the field’s data type. If the field is an integer, then we must supply an integer value to the parameter. Lastly, the parameter influences how data is indexed, meaning the data structures that are used for searches. The `_source` object is therefore not affected, and will contain the value that was supplied when indexing the document, i.e. `NULL`.

```json
PUT /sales
{
    "mappings": {
        "properties": {
            "partner_id": {
                "type": "keyword",
                "null_value": "NULL"
            }
        }
    }
}
```

#### `copy_to` parameter
Suppose that we have the name of a person stored as two fields; `first_name` and `last_name`. While we can query one or both of these fields just fine, we might also want to index the full name into a third field, perhaps named `full_name`: that’s possible with the `copy_to` parameter.

The value for the parameter should be the name of the field to which the field value should be copied. So by adding the parameter to both the “first_name” and “last_name” fields, Elasticsearch will copy both values into the “full_name” field. Notice how I said that values will be copied. That’s because it’s not the tokens or terms yielded by the analyzer that are copied; instead, it’s the values that were supplied. These values are then analyzed with the analyzer of the target field.


``` json
PUT /sales
{
    "mappings": {
        "properties": {
            "first_name": {
                "type": "text",
                "copy_to": "full_name"
            },
            "last_name": {
                "type": "text",
                "copy_to": "full_name"
            },
            "full_name": {
                "type": "text"
            }
        }
    }
}
```

One important thing to note is that the copied values will not be part of the `_source` object. In this example it means that the “full_name” field is not part of the “_source” object, since the values are only indexed. The exception to that is if you specify a value for the “full_name” field when you index the document, but that’s not a very common thing to do at all.

```json
POST /sales/_doc
{
    "first_name": "John",
    "last_name": "Doe"
}
```

### Updating existing mappings
We know it's possible to add a new field mappings to an existing index. However, except some cases, we cannot update an existing field mapping.

Some mapping parameters can be updated, but only a few of them. For instance, if we have a field mapping we can add the parameter `ignore_above` to ignore strings longer than the specified value such that they will not be indexed or stored.

```json
PUT /reviews/_mapping
{
    "properties": {
        "author": {
            "properties": {
                "email": {
                    "type": "keyword",
                    "ignore_above": 256
                }
            }
        }
    }
}
```

We also cannot remove a field mapping once you have added it. If we want to reclaim the storage space used by the field, we could use the `update_by_query` API and remove the field from each matching document with a script.

Changing a field mapping almost always requires documents to be reindexed. That’s why Elasticsearch protects us by simply not allowing existing mappings to be updated - except for a small number of parameters.

### `reindex` api

To reindex documents we have to create a new index and define the new mapping and optionally any index settings. To edit the existing mapping it can be useful retrieve the current one

```
GET /reviews/_mappings
```

Let's create the new index with the new mapping

```json
PUT /reviews_new
{
    "mappings": {
        "properties": {
            ...
        }
    }
}
```

Once the new index has been created, to reindex the documents from the previous index to the new one elasticsearch expose the `reindex` api. The HTTP verb is `POST` and the endpoint is simply `_reindex`

```json
POST /_reindex
{
    "source": {
        "index": "source_index"
    },
    "dest": {
        "index": "destination_index"
    }
}
```

We can modify documents during reindexing, is to supply a script that is run for each document.

```
POST /_reindex
{
    "source": {
        "index": "reviews"
    },
    "dest": {
        "index": "reviews_new"
    },
    "script": {
        "source": """
        if(ctx._source.product_id != null){
            ctx._source.product_id = ctx._source.product_id.toString();
        }
        """
    }
}
```

To delete all the documents in an index, we can use the `delete_by_query` api: 

```
POST /reviews_new/_delete_by_query
{
    "query": {
        "match_all": {}
    }
}
```

We can reindex documents that only matches a query

```
POST /_reindex
{
  "source": {
    "index": "reviews",
    "query": {
      "range": {
        "rating": {
          "gte": 4.0
        }
      }
    }
  },
  "dest": {
    "index": "reviews_new"
  }
}
```


Next, suppose that we want to remove a field: we cannot delete a field mapping, but we can just leave out the field when indexing new documents. However, if we have indexed lots of documents, perhaps we want to reclaim the disk space used by the field. Remember that even though we stop supplying a value for a field when indexing new documents, the existing values are still maintained within a data structure.
The way we can do this is to use so-called **source filtering**. By specifying an array of field names, only those fields are included for each document when they are indexed into the destination index. In other words, any fields that you leave out will not be reindexed. It’s possible to do the same thing with a script, but using the `_source` parameter is a simpler approach.

```
POST /_reindex
{
  "source": {
    "index": "reviews",
    "_source": ["content", "created_at", "rating"]
  },
  "dest": {
    "index": "reviews_new"
  }
}
```


Sometimes you might want to rename a field, such as renaming the `content` field to `comment`. Just to know, the `_source` key translates to a Java HashMap. 


```
POST /_reindex
{
  "source": {
    "index": "reviews"
  },
  "dest": {
    "index": "reviews_new"
  },
  "script": {
    "source": """
      # Rename "content" field to "comment"
      ctx._source.comment = ctx._source.remove("content");
    """
  }
}
```


Just as with scripted updates, it’s possible to specify the operation for the document within the script. That’s done by assigning a value to the “op” property on the `ctx` variable. We can assign a value of either `noop` or `delete`. In the case of `noop` the document will not be indexed into the destination index. This is useful if you have some advanced logic to determine whether or not you need to reindex documents.

```
POST /_reindex
{
  "source": {
    "index": "reviews"
  },
  "dest": {
    "index": "reviews_new"
  },
  "script": {
    "source": """
      if (ctx._source.rating < 4.0) {
        ctx.op = "noop"; # Can also be set to "delete"
      }
    """
  }
}
```

Setting the operation to `delete` causes the document to be deleted within the destination index. This can be useful because the document might already exist within that index.

`reindex` api has more options. For example, you can define how to handle version conflicts, just as with the Update by Query API. That’s because the Reindex API also makes a snapshot of the index before indexing documents into the destination index. By default, the operation is aborted if a version conflict is encountered.


### Field Aliases
We can define field aliases that point to other existing fields. Field aliases can be used in our queries instead of the original name.

Add `comment` alias pointing to the `content` field using the `path` parameter

```
PUT /reviews/_mapping
{
  "properties": {
    "comment": {
      "type": "alias",
      "path": "content"
    }
  }
}
```

Now we can use the alias `comment` in place of the field `content`

```
GET /reviews/_search
{
  "query": {
    "match": {
      "comment": "outstanding"
    }
  }
}
```

A field alias is useful in situations where you want to rename a field but don’t want to reindex data just for that purpose.

A field alias is actually one of the few examples of mappings that can be updated. Not its name, but its target field. If you want to change the field that an alias points to, you can simply perform a mapping update with a new value for the `path` parameter. That’s possible because an alias has no influence on how values are indexed; it is simply a construct used when parsing queries, whether it’s a search or index request.


### Multi field mappings
A field may actually be mapped in multiple ways. For instance, a `text` field may be mapped as a `keyword` field at the same time. We cannot run aggregations on `text` fields, we need to use the `keyword` data type. This is why can be useful to have a field with multiple types. Aggregations can also be run on dates and numbers. 

```
PUT /multi_field_test
{
  "mappings": {
    "properties": {
      "description": {
        "type": "text"
      },
      "ingredients": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```


### Index Templates
An index template is a way to automatically apply settings and field mappings whenever a new index is created - provided that its name matches one of the defined patterns. 

To create a new index template, we send a PUT request to the request path you see within the example, where the last part is an arbitrary name for the index template. The index_patterns option is an array of wildcard expressions that are used to match against the name of any new index. Before creating a new index, Elasticsearch checks if there are any index templates containing at least one pattern that matches the index name. If so, the index template is applied to the new index’ configuration before it is created. The index_patterns option therefore configures when the index template is applied. Note that the two nested keys are optional, so you could choose to only include index settings, for instance.

```json
PUT /_index_template/access-logs
{
  "index_patterns": ["access-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "index.mapping.coerce": false
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "url.original": { "type": "wildcard" },
        "url.path": { "type": "wildcard" },
        "url.scheme": { "type": "keyword" },
        "url.domain": { "type": "keyword" },
        "client.geo.continent_name": { "type": "keyword" },
        "client.geo.country_name": { "type": "keyword" },
        "client.geo.region_name": { "type": "keyword" },
        "client.geo.city_name": { "type": "keyword" },
        "user_agent.original": { "type": "keyword" },
        "user_agent.name": { "type": "keyword" },
        "user_agent.version": { "type": "keyword" },
        "user_agent.device.name": { "type": "keyword" },
        "user_agent.os.name": { "type": "keyword" },
        "user_agent.os.version": { "type": "keyword" }
      }
    }
  }
}
```

However, even if an index matches a template, the settings specified at the index creation have the precedence before the ones specified in the template

To update a template the request is exactly the same (note the `PUT` verb). This doesn’t modify indices that have already been created.

To retrieve an index template

```
GET /_index_template/access-log
```

To delete an index template

```
DELETE /_index_template/access-log
```

When defining your index patterns, it’s important to know that Elasticsearch ships with a handful of index templates that are used for its observability solution. We should therefore stay clear of using index patterns that collide with these. If that’s not an option, you can disable these index templates or circumvent them by adding priorities to your index templates. Check [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-template.html) for more information. The reserved index patterns are

```
logs-*-*
metrics-*-*
synthetics-*-*
profiling-*
```

By default you cannot add multiple index templates that contain overlapping index patterns. That’s because only a single index template can be applied to a new index. If you try to add a new index template that contains an index pattern that overlaps with another one, you will get an error.

To solve this, we can add a priority option to both index templates. The value should be a natural number, i.e. zero or higher. When no explicit priority is specified, it defaults to zero. The higher the number, the higher priority an index template has. The two index templates can therefore be added despite the fact that their index patterns overlap. Whenever multiple index templates match a new index, the one with the highest priority is simply used.


### Introduction to dynamic mapping

It’s essentially a way to make Elasticsearch easier to use by not requiring us to define explicit field mappings before indexing documents. The first time Elasticsearch encounters a field, it will automatically create a field mapping for it, which is then used for subsequent indexing requests.

When a document with unknown fields is inserted in an index, elasticsearch automatically creates a mapping for each of such fields. Let's suppose we have created an index with no mapping. If we add a document like the following

```
POST /dynamic_index/_doc
{
  "tags": ["computer", "electronics"],
  "in_stock": 4,
  "created_at": "2020/01/01 00:00:00"
}
```

The mapping we get is

```
{
  "created_at":  {
    "type": "date",
    "format": "yyyy/MM/dd HH:mm:ss|| yyyy/MM/dd||epoch_millis"
  },
  "in_stock": {
    "type": "long",
  },
  "tags": {
    "type": "text",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256,
      }
    }
  }
}
```

where the date has been marked as date even if it was provided as a string because elasticsearch performs a date detection. The `tags` field was mapped as a multi-field with both the `text` and `keyword` data types. This is because elasticsearch doesn't know if we will use the field to perform text queries or aggregations. 

Elasticsearch has riles that are used to determine how fields are mapped dynamically

| JSON | Elasticsearch |
| ------  | -------------- |
| string  | one of the following: `text` field with `keyword` mapping, `date` field, `long` or `float` field |
| integer                | `long` |
| floating point numbers | `float` |
| true or false          | `boolean` |
| object                 | `object` |
| array                  | depends on the first non null value | 

Dynamic mapping does not always infer the right mapping.


### Configuring dynamic mapping

We can disable dynamic mapping when creating an index

```
PUT /people
{
  "mappings": {
    "dynamic": false,
    "properties": {
      "first_name": {
        "type": "text"
      }
    }
  }
}
```

Now if we try to add a document to the index 

```
POST /people/_doc
{
  "first_name": "Bo",
  "last_name": "Andersen"
}
```

We don't get any error, but if we try to perform a query based on the element not present in the mapping, no results will be returned. So, the following will work and we get the whole document with both `first_name` and `last_name`

```
GET /people/_search
{
  "query": {
    "match": {
      "first_name": "Bo"
    }
  }
}
```

Instead, the following won't give us back any document

```
GET /people/_search
{
  "query": {
    "match": {
      "last_name": "Andersen"
    }
  }
}
```

In other words, the fields not present in the mapping do not trigger any data structured associated to perform queries. 


This approach works but it is not recommended. `dynamic` can not be set only to `true` or `false` but also to `strict`. With this last configuration, elasticsearch will reject any document that contain fields not present in the index mapping.

```
PUT /people
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "first_name": {
        "type": "text"
      }
    }
  }
}
```


Inheritance is actually supported for the `dynamic` setting, which gives you fine grained control of dynamic mapping. Suppose that we are indexing computers. We set dynamic mapping to `strict` at the index level. All of the fields within the index inherit this configuration. At the top level, this means that both the `name` and `specifications` fields inherit the `dynamic` setting with a value of `strict`. The inheritance works on multiple levels, so the nested `cpu` and `other` fields inherit the setting from its parent field, being the `specifications` field. Since the `cpu` field has inherited strict mapping, we can only provide a `name` field for this object. 

```
PUT /computers
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "text"
      },
      "specifications": {
        "properties": {
          "cpu": {
            "properties": {
              "name": {
                "type": "text"
              }
            }
          },
          "other": {
            "dynamic": true,
            "properties": { ... }
          }
        }
      }
    }
  }
}
```

The index query on your screen therefore fails because there is no mapping for the `frequency` field. The `other` field stores many different fields depending on the product, so we cannot map all of the possible fields in advance. Instead, we can enable dynamic mapping for just that particular field, overriding the inherited value. Since the `other` field is an object, its properties will inherit this configuration, enabling us to add new fields dynamically. In this example, we can add a `security` field just fine.

The following query hence will fail

```
POST /computers/_doc
{
  "name": "Gamer PC",
  "specifications": {
    "cpu": {
      "name": "Intel Core i7-9700K",
      "frequency": 3.6
    }
  }
}
```

Instead, this one will work

```
POST /computers/_doc
{
  "name": "Gamer PC",
  "specifications": {
    "cpu": {
      "name": "Intel Core i7-9700K"
    },
    "other": {
      "security": "Kensington"
    }
  }
}
```

We can enable numeric_detection: elasticsearch will check the contents of strings to see if they contain only numeric values. If that is the case, the data type of a field witll be set to either `float` or `long` when mapped through dynamic mapping.

The mapping

```
PUT /computers
{
  "mappings": {
    "numeric_detection": true
  }
}
```

A document example

```
POST /computers/_doc
{
  "specifications": {
    "other": {
      "max_ram_gb": "32", # long
      "bluetooth": "5.2" # float
    }
  }
}
```

We can also configure **date_detection**. By default, Elasticsearch inspects string values to look for dates in one of the formats: `strict_date_optional_time`, `yyyy/MM/dd HH:mm::ss Z` or `yyyy/MM/dd Z`. If there is a match, Elasticsearch will create a `date` field, provided that the field hasn’t been seen before, and that dynamic mapping is enabled. The field is created using the default date format. 

We can disable date detection altogether by setting the `date_detection` setting to `false`

```
PUT /computers
{
  "mappings": {
    "date_detection": false
  }
}
```

We can also configure the dynamic date formats that are recognized. This may be useful if you have applications sending dates to Elasticsearch in non-standard formats. Those were the basic — and probably most common — ways in which you can configure dynamic mapping. 

```
PUT /computers
{
  "mappings": {
    "dynamic_date_formats": ["dd-MM-yyyy"]
  }
}
```

There is another way, though, described in the next paragraph.


### Dynamic Templates
Another way to configure dynamic mapping is by using so-called **dynamic templates**. A dynamic template consists of one or more conditions along with the mapping a field should use if it matches the conditions. Dynamic templates are used when dynamic mapping is enabled and a new field is encountered without any existing mapping.

Dynamic templates are added within a key named `dynamic_templates`, nested within the `mappings` key. Let's define a dynamic template that make elasticsearch assign an `integer` type instead of a `long` type

```json
PUT /dynamic_template_index
{
  "mappings": {
    "dynamic_templates": {
      "integers": {
        "match_mapping_type": "long"
      }
    }
  }
}
```

The `match_mapping_type` parameter is used to match a given JSON data type. The following table lists JSON data types and which value should be used for the parameter within the dynamic template. The values to the right are not the data types that fields will be mapped as; we will define those in a moment. Moreover, the  `double` is used for numbers with a decimal, since there is no way of distinguishing a `double` from a `float` in JSON, the same for `integer` and `long`. The condition is specified within the `mapping` parameter. Hence, we have defined a condition that evaluates to true if the detected JSON data type is `long`

in which case the field mapping will be as specified.

| JSON value        | `match_mapping_type` |
| :---------------: | :------------------: |
| true or false     | `boolean`        |
| { ... } (objects) | `object`         |
| "string value"    | `string`         |
| "2020/01/01"      | `date`           |
| 123.4             | `double`         |
| 123               | `long`           |
| Any               | `*`              |


Another example of a dynamic template could be to adjust how strings are mapped.They are mapped as both a `text` mapping and a `keyword` mapping by default. That might be what you want, but it could also be the case that you only want one of them. Or perhaps you want to change the value of the `ignore_above` parameter or get rid of it altogether.

```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 512
              }
            }
          }
        }
      }
    ]
  }
}
```

Besides simply matching based on the detected JSON data type, we have a number of other options available, like the `match` and `unmatch` parameters.  These parameters enable to specify conditions for the name of the field being evaluated. The `match` parameter is used to specify a pattern that the field name must match, `unmatch` parameter can then be used to specify a pattern that excludes certain fields that are matched by the `match` parameter.

Considering the following, it includes two dynamic templates. In situations where there are more than one, the templates are processed in order, and the first matching template wins.

```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_only_text": {
          "match_mapping_type": "string",
          "match": "text_*",
          "unmatch": "*_keyword",
          "mapping": {
            "type": "text"
          }
        }
      },
      {
        "strings_only_keyword": {
          "match_mapping_type": "string",
          "match": "*_keyword",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}
```

The first template matches fields where the name begins with the string `text` and an underscore. The `unmatch` parameter when filters out fields that end with an underscore and the string `keyword`. What this template does is therefore to map string values to `text` fields — provided that the field name matches the pattern defined by the `match` parameter and doesn’t match the pattern defined by the `unmatch` parameter.

Fields names that end with an underscore followed by the string `keyword` are caught by the second template that maps them as `keyword` fields. This means that if we run the following query the description field matched the first template, whereas the product ID field matched the second.

```json
POST /test_index/_doc
{
  "text_product_description": "A description.",
  "text_product_id_keyword": "ABC-123"
}
```

Both the templates included the `match_mapping_type` parameter as well because we can include more than one condition within a dynamic template; the parameters that you have seen can all be combined to form a set of match conditions, so you are not limited to just one.

If we need more flexibility than what the `match` parameter provides with wildcards, we can set a parameter named `match_pattern` to `regex`.

```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "names": {
          "match_mapping_type": "string",
          "match": "^[a-zA-Z]+_name$",
          "match_pattern": "regex",
          "mapping": {
            "type": "text"
          }
        }
      }
    ]
  }
}
```

We can test the dynamic template with the following query

```json
POST /test_index/_doc
{
  "first_name": "John",
  "middle_name": "Edward",
  "last_name": "Doe"
}
```

Very similar to the `match` and `unmatch` parameters, we have the `path_match` and `path_unmatch` parameters. The difference is that these parameters match the full field path (i.e. the dotted path), instead of just the field name itself.
Let’s look at an example.

```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "copy_to_full_name": {
          "match_mapping_type": "string",
          "path_match": "employer.name.*",
          "mapping": {
            "type": "text",
            "copy_to": "full_name"
          }
        }
      }
    ]
  }
}
```

the `copy_to` mapping parameter, which copies values into the specified field. In this case it enables us to query the full name as a single field, instead of having to query three different fields. A mapping will be created for the `full_name` field. It will be created using the default dynamic mapping rule for string values, meaning that it will be mapped as both a `text` and `keyword` field.

Within the `mapping` key of a dynamic template, you can make use of two placeholders. The first one is named `dynamic_type`. The `dynamic_type` placeholder is replaced with the data type that was detected by dynamic mapping and matches all data types and adds a mapping of that same data type.

```json
PUT /test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "no_doc_values": {
          "match_mapping_type": "*",
          "mapping": {
            "type": "{dynamic_type}",
            "index": false
          }
        }
      }
    ]
  }
}
```

The purpose of this template is to set the `index` parameter to `false`. The data type is going to be the same as it otherwise would be with dynamic mapping. This particular example with disabling indexing could be used for time series data.

Anyway, let’s check the mapping that is created when adding the following document. As you can see, both field mappings have the `index` parameter set to `false` as we expected.

```json
POST /test_index/_doc
{
  "name": "John Doe",
  "age": 26
}
```

The difference between an index template and a dynamic template is that an index template applies field mappings and/or index settings when its pattern matches the name of an index. A dynamic template, on the other hand, is applied when a new field is encountered and dynamic mapping is enabled. If the template’s conditions match, the field mapping is added for the new field. A dynamic template is therefore a way to dynamically add mappings for fields that match certain criteria, where an index template adds a fixed set of field mappings.


## Searching for Data

### Introduction
There are two ways in which we can write search queries. The first one is by including search queries as a query parameter, which is referred to as a URI search. The value of the query parameter should be in Apache Lucene’s query string syntax. The syntax won't be covered in this course because it is rarely used because it’s a simplified search that doesn’t offer all of the features that Elasticsearch provides. Plus, you would have to get familiar with Lucene’s query string syntax.

```
GET /products/_search?q=name:sauvigon AND tags:wine
```

where `q=name:sauvigon AND tags:wine` is the Apache Lucene syntax.


Instead, we will cover `Query DSL`, which is the preferred way of writing search queries. Instead of embedding the query within a query parameter, it’s added within the request body, meaning that it should be in JSON.
The two approaches use the same request path — and thereby API — being the Search API. The search query itself is defined within an object named `query`, in which we define the type of query as a key. The simplest query is the `match_all` query

```dsl
GET /products/_search
{
  "query": {
    "match_all": {}
  }
}
```

A possible is the following, where

- `took` tells the time it took for Elasticsearch to execute the request from the time it was received, in ms
- `timed_out` tells whether or not the request timed out
- `_shards`
  - `total` specifies how many shards should be queried to execute the request. Note that this number includes shards that haven’t been allocated, so if the number is higher than you expect, then that is probably the reason.
  - `successful` and `failed` specify the number of shards that executed the request successfully or unsuccessfully
  - `skipped` represents the number of shards that skipped the request. A shard may skip executing the request if it determines that it contains no documents that can possibly match the request. This typically occurs for range values where the shard only stores documents that fall outside of the specified range.
- `hits` contains the documents that matched the query along with some metadata. We have a total key that shows us how many documents matched the query.
  - `eq` if the value key contains an accurate number, which will almost always be the case. There is a way to do some performance tuning of queries at the tradeoff of the value not being accurate. In that case the key will contain a value of `gte` 
  - `max_score` contains the highest document score that was calculated by Elasticsearch.
  -`hits` contains the actual documents that matched the query along with a bit of metadata for each document. First, we have the name of the index in which the document is stored.
    -`_ignored` contains the names of fields that were ignored when the document was indexed. For this document that was the description.keyword field. The reason is that the mapping contains the ignore_above parameter with a value of 256, meaning that values longer than this are ignored. That’s how text fields are mapped by default with dynamic mapping.
    -`_source` is the actual document

```dsl
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 2,
    "successful": 2,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1000,
      "relation": "eq",
    },
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 1,
        "_ignored": [
          "description.keyword"
        ],
        "_source": {
          "name": "Wine - Maipo Valle Cabernet",
          "price": 152,
          "in_stock": 38,
          "sold": 47,
          "tags": [
            "Beverage"
            "Alcohol"
          ],
          "description": "sample description",
          "is_active": true,
          "created": "2004/05/13"
        },
        {
          ...
        }
    ]
  }
}
```








