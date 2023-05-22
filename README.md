
test 14

(unofficial)
# Introduction to using the Elasticsearch Python client:

<br />

# Motivation for this repository:
There exists a great
[Beginner's Crash Course to Elastic Stack](https://github.com/LisaHJung/Beginners-Crash-Course-to-Elastic-Stack-Series-Table-of-Contents) 
which includes the 
[Beginner's Crash Course to Elastic Stack workshop playlist](https://www.youtube.com/playlist?list=PL_mJOmq4zsHZYAyK606y7wjQtC0aoE6Es)
where, above else, they introduce the basic operations of the [Elastic Search REST API](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/rest-apis.html).

There also exist many 
[language-specific clients](https://www.elastic.co/guide/en/elasticsearch/client/index.html) 
for ElasticSearch (which use this REST API under the hood), but
it can be difficult to figure out how exactly the original REST API commands 
correspond to the language-specific commands.

**So this repository gives an introduction to the most basic Python Client commands for ElasticSearch.**

---

<br /><br />

python elasticsearch client docs: https://elasticsearch-py.readthedocs.io/

(note: I could mix the commands of ElasticSearch versions 7 and 8 here, didn't pay much attention to that)

<br />

#### **BONUS**: 

Consider a useful trick:

> When you see a REST API command but don't know its equivalent in Python, e.g. 
> this one: https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html),
> how to find out the python command for it?

Search for back-reference:

- copy the last part of your url: `indices-put-mapping.html`
- use this search on Google: <br />
`site: https://elasticsearch-py.readthedocs.io/ "indices-put-mapping.html"` where you insert your copied text in the double quotes.
- This will find this exact text on the website of the Python API. It's convenient that the Python API website usually links the link to the REST API equivalent of each command. So this lets us find the webpage.
- On this webpage use `ctrl+f` to find the exact position of `indices-put-mapping.html`

---


Alternatively, you can avoid using the Python Client completely, and just use the 
original REST API from Python with the help of `requests` library.


---


<br /><br />

## The Python Client basic TUTORIAL:


```python
# Make sure the package `elasticsearch` is installed:
%pip install -U elasticsearch
```

<br />


```python
# Import:
from elasticsearch import Elasticsearch

# optional - for pretty printing:
import pprint
pp = pprint.PrettyPrinter()
```


```python
# Assumming the ElasticSearch database is already running on localhost:9200,
# Instance of the Elasticsearch client:
es = Elasticsearch('localhost:9200')

# Alternatively,
# es = Elasticsearch(['localhost'], port=9200)
```


```python
# List all indices in a cluster:
print(es.cat.indices(v=True, s='health'))  # v=True -> show headings, s='health' -> sort by health
```

<br />


```python
# CRUD operations:
index_name = "my-test-index"

# ---
# create an index
es.indices.create(index=index_name, ignore=400)

# add a document
es.index(index=index_name, id=1, body={"field_1": "value1", "field_2": 2})

# ---
# get a document
es.get(index=index_name, id=1)

# ---
# update a document
es.update(index=index_name, id=1, body={'doc': {'field_1': 'updated1'}})

# ---
# delete a document
es.delete(index=index_name, id=1)

# delete an index
es.indices.delete(index=index_name)
```


```python
# Notes:

# When creating an index you can also already specify its mapping (and other settings that go into `body`):
mapping = {
    "properties": {
        "field_1": {"type": "text"},
        "field_2": {"type": "integer"}
    }
}
es.indices.create(index=index_name, body={"mappings": mapping})


# When creating an index:
# ignore=400 ignores the 400 cause by IndexAlreadyExistsException when creating an index.
# in v.7.x - here's the explanation of the `ignore` parameter:
# https://elasticsearch-py.readthedocs.io/en/7.x/api.html?highlight=elasticsearch.indices#ignore
# in v.8.6.2 - could not find the `ignore` parameter in the docs:
# https://elasticsearch-py.readthedocs.io/en/v8.6.2/api.html#elasticsearch.client.IndicesClient.create


# When indexing a document (i.e., adding a document to the index):
es.index(index=index_name, id=1, body={"field_1": "value_1", "field_2": 2})
# es.create could also work (instead of es.index), but there are some differences. 
# With es.create:
# - if the doc with the given id already exists, it will raise an error. 
# - es.create requires an id, while with es.index we don't have to specify the id, it will be auto-generated (alpha-numeric).

```

<br />


```python
# See the mapping of an index:
print('-- mapping:')
pp.pprint(
    es.indices.get_mapping(index=index_name)
)
print()

# See the specific document(s) of an index:
print('-- document 1:')
pp.pprint(
    es.get(index=index_name, id=1)
)
print()

# See ALL the documents of an index:
print('-- all documents:')
pp.pprint(
    es.search(index=index_name, body={"query": {"match_all": {}}})
)
# can even omit the `body`:
# es.search(index=index_name, size=4)
print()

# Do the aggregations:
data = es.search(index=index_name,
                 body={
                    'size': 0,  # to not return any actual documents
                    'aggs': {
                        'result_fieldname_min': {'min': {'field': 'existing_field_name'}},
                        'result_fieldname_max': {'max': {'field': 'existing_field_name'}},
                    }
                 })
data['aggregations']
```
```

<br />


```python
# # Copy data from an existing index into a new index:
new_index_name = "my-new-index"
es.reindex(
    max_docs=2,  # limit: only copy the 2 first documents
    body={
        "source": {
            "index": index_name  # source index
        },
        "dest": {
            "index": new_index_name  # destination index
        }
    })



# For the NEW fields in an index - can put the mapping before adding document(s) -
# (to have specific control over the types of these fields):
mapping = {
    "properties": {
        "new_field": {"type": "text"}
    }
}
es.indices.put_mapping(
    index=new_index_name,
    body=mapping
)

# can also index a document like this:
es.update(
    index=new_index_name,
    id=2,
    body={
        "doc": {
            "field_1": "value_1_updated",
            "new_field": "test value 2"
        },
        "doc_as_upsert": True 
            # "doc_as_upsert" means that in case this id (id=2) does not exist in the index - 
            # create a document with this id.
            # If "doc_as_upsert" wasn't specified - it would be False by default, 
            # and an error would be raised in the above case.
    }   
)

```

<br />

### How to read data from ElasticSearch to a Pandas dataframe?


```python
import pandas as pd
from pandas import json_normalize

data = es.search(index=index_name,
                 size=2,  # return only 2 documents (for speed purposes)
                 body={
                    "query": {"match_all": {}}
                 })

df = json_normalize([dict(x['_source'], **{'_id': x['_id']}) for x in data['hits']['hits']]).set_index('_id')
```

<br />

#### More advanced (no limit of 10_000 docs):
(ES version 7.13)


```python
def read_ES_to_df(es: Elasticsearch,
                start_ms: int, 
                end_ms: int,
                max_read_limit: int = 10_000) -> pd.DataFrame:
"""
Read the data from Elasticsearch database (from the index `my_index`)
that is between the given timestamps into a pandas DataFrame.
By default, the max. number of results allowed to be returned from ES is 10_000.

Parameters
----------
es : Elasticsearch
    ES object.
start_ms : int
    Start datetime: number of milliseconds since the epoch.
end_ms : int
    End datetime: number of milliseconds since the epoch.
max_read_limit : int, optional
    Max. number of results allowed to be returned from ES, as
    a safety measure. The default is 10_000.
    If -1, then no limit is applied (side-effect: ES might crash if used 
    without caution)!

Returns
-------
df : pd.DataFrame
    DataFrame with the data from ES, between the given timestamps.
"""
index_name = 'my_index'

# helper function
def _ES_data_to_df(data: dict) -> pd.DataFrame:
    """
    Convert the JSON response from ES search to a DataFrame.

    Parameters
    ----------
    data : dict
        Data from ES search query.
    
    Returns
    -------
    df : pd.DataFrame
        DataFrame containing all the data from `_source`, 
        and with ids from `_id`.
    """
    df = pd.json_normalize([dict(hit['_source'], **{'_id': hit['_id']})
                            for hit in data['hits']['hits']]).set_index('_id')
    return df

# Note:
# Avoid using `size` parameter to request too many results at once, because:
# - Search requests usually span multiple shards. Each shard must load its requested 
#   hits and the hits for any previous pages into memory. 
#   For deep pages or large sets of results, these operations can significantly 
#   increase memory and CPU usage, resulting in degraded performance or node failures.
# By default, you cannot use `size` to page through more than 10,000 hits.
# This limit is a safeguard. There are 2 options to work around it:

# 1) If you need to page through more than 10,000 hits, the recommended
#   way is to use the `search_after` parameter, with a point in time! (PIT)
#   https://www.elastic.co/guide/en/elasticsearch/reference/7.13/paginate-search-results.html#search-after
#   https://www.elastic.co/guide/en/elasticsearch/reference/7.13/point-in-time-api.html#point-in-time-api-example
#   - Keeping older segments alive means that more disk space and file handles are needed. 
#     Ensure that you have configured your nodes to have ample free file handles. See File Descriptors.
#   - Ensure that your nodes have sufficient heap space if you have many open point-in-times on an index that 
#     is subject to ongoing deletes or updates.

# 2) (not recommended)
#   While a search request returns a single “page” of results, 
#   the `scroll` API can be used to retrieve large numbers of results 
#   (or even all results) from a single search request, in much the 
#   same way as you would use a cursor on a traditional database:
#   https://www.elastic.co/guide/en/elasticsearch/reference/7.13/paginate-search-results.html#scroll-search-results
#   Scrolling is not intended for real time user requests, but rather 
#   for processing large amounts of data, e.g. in order to reindex the 
#   contents of one data stream or index into a new data stream or 
#   index with a different configuration.
#   The point-in-time API supports a more efficient partitioning strategy.
#   When possible, it’s recommended to use a point-in-time search 
#   with slicing instead of a scroll.


# Search the database, filtering it by timestamps
body = {
    'query': {
        'bool': {
            'filter': [
                {'range': {'time_ms': {'gte': start_ms, 'lte': end_ms}}}
            ]
        }
    },
    "sort": [{"time_ms": "asc"}]  # (not necessary here)
}

if max_read_limit <= 10_000 and max_read_limit != -1:
    # Normal search:
    body["size"] = max_read_limit
    data = es.search(index=index_name, body=body)
    df = _ES_data_to_df(data)

else:
    # Use the `search_after` parameter, with a point in time (PIT):
    try:  # (makes sure the PIT is closed in case error happens)
        keep_alive = '2m'  # time to keep the PIT alive for  # TODO: adjust this time yourselves
        pit = es.open_point_in_time(index=index_name, keep_alive=keep_alive)
        pit_id = pit['id']
        body["pit"] = {"id": pit_id, "keep_alive": keep_alive}
        body["size"] = 10_000
        body["sort"] = [{"time_start_ms": "asc"}]  # sorting is required for pagination
        total_read = 0  # total number of results (rows) read so far
        partial_dfs = []
        while True:
            data = es.search(body=body)  # note: since we're using PIT, we MUST NOT specify the index name here
            read_now = len(data['hits']['hits'])
            if read_now == 0:
                break  # all results have been read
            partial_dfs.append(_ES_data_to_df(data))

            total_read += read_now
            left_to_read = max_read_limit - total_read
            if left_to_read <= 0:
                break  # max. allowed number of results has been read
            if left_to_read < 10_000:
                # Next iteration will be the last one. Read only the remaining results:
                body["size"] = left_to_read
            pit_id = data['pit_id']  # must always use the most recently received PIT id for the next search request
            body["pit"]["id"] = pit_id
            body["search_after"] = data['hits']['hits'][-1]['sort']  # the "Sort values" of the last returned hit, for pagination.
    finally:
        es.close_point_in_time(body={'id': pit_id})  # close the PIT
    df = pd.concat(partial_dfs)
return df
```

<br />

### How to save a dataframe to ElasticSearch?


```python
df1 = pd.DataFrame({
    "field_1": ["aa", "bb", "cc"],
    "field_2": [1, 2, 3]
})
```


```python
# bulk update
from elasticsearch.helpers import bulk

# All bulk helpers accept an instance of `Elasticsearch` class and an iterable `actions` 
# (any iterable -- it can also be a GENERATOR, which IS IDEAL in most cases -
# since it will allow you to index large datasets without the need of loading them into memory all at once).
# https://elasticsearch-py.readthedocs.io/en/v7.17.9/helpers.html?highlight=update#elasticsearch.helpers.bulk
# https://towardsdatascience.com/exporting-pandas-data-to-elasticsearch-724aa4dd8f62 (example)


def filterKeys(row):
    # If the value of a field is na, it will not be added to the document:
    # (if v is a sequence, include it, even if it consists only of nans)
    return {k: v for k, v in row.items() if hasattr(v, "__len__") or not pd.isna(v)}

def doc_generator(df):
    for idx, row in df.iterrows():
        doc = {
            # "_op_type": "index",  # by default, it's "index". Can also be "create", "delete", "update"
            '_index': index_name,
            '_type': '_doc',
            "_id" : idx,
            "_source": filterKeys(row),  # {"field_1": "aa", "field_2": 1}
        }
        yield doc

bulk(es, doc_generator(df1))
```
