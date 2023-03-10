# Introduction to the Elasticsearch Python client: <br /> https://elasticsearch-py.readthedocs.io/

(unofficial) <br />
(warning: commands of Elasticsearch versions 7 and 8 could be both blended in here, but I mainly tried to use v. 8)

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

<br />

#### **BONUS**: 

Consider a useful trick:

> When you see a REST API command but don't know it's equivalent in Python, e.g. 
> this one: https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html,
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

<br />


```python
# # Copy data from an existing index into a new index:
new_index_name = "my-new-index"
es.reindex(
    max_docs=2,  # limit: only copy 2 documents
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



