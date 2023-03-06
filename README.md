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
> [this one](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html),
> how to find out the python command for it?

Search for back-reference:

- copy the last part of the url: `indices-put-mapping.html`
- use this search on Google: <\n>
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

# When creating an index, you can also already specify its mapping (and settings, etc. in the `body`):
mapping = {
    "properties": {
        "field_1": {"type": "text"},
        "field_2": {"type": "integer"}
    }
}
es.indices.create(index=index_name, body={"mappings": mapping})


# When creating an inde:
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
print()
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

# can index a document:
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


```python
# How to send your dataframe to ElasticSearch?

# new data in a Pandas dataframe:
import pandas as pd
df1 = pd.DataFrame({
    "field_1": ["aa", "bb", "cc"],
    "field_2": [1, 2, 3]
})
# A dataframe with the new fields to be added to the documents:
df2 = pd.DataFrame({
    "field_3": ["a", "b", "c"],
    "field_4": [4, 5, 6]
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
    return {k: v for k, v in row.items() if v is not None}

def doc_generator(df):
    df_iter = df.iterrows()
    for idx, row in df_iter:
        doc = {
            # "_op_type": "index",  # by default, it's "index". Can also be "create", "delete", "update"
            '_index': index_name,
            '_type': 'document',
            "_id" : idx,
            "_source": filterKeys(row),  # {"field_1": "val1", "field_2": 2312}
        }
        yield doc

bulk(es, doc_generator(df1))

```


```python

```
