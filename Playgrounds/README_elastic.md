
# Elasticsearch queries

Repasemos las posibles queries que ofrece elastic, desde las mas comunes hasta algunas mas complejas.  

## Setup

Lo primero es declarar el cliente y un par de metodos para facilitar los ejemplos.


```python
from elasticsearch import Elasticsearch
from dateutil.parser import parse as parse_date


es = Elasticsearch("http://elasticsearch:9200")
es.info()

def print_hits(results):
    " Simple utility function to print results of a search query. "
    print_search_stats(results)
    for hit in results['hits']['hits']:
        # get created date for a repo and fallback to authored_date for a commit
        print('/%s/%s/%s [%s]: |%s| %s - %s' % (
                hit['_index'], hit['_type'], hit['_id'], hit['_score'],
                hit['_source']['play_name'].split('\n')[0],
                hit['_source']['speaker'].split('\n')[0],
                hit['_source']['text_entry']))

    print('=' * 80)
def print_search_stats(results):
    print('=' * 80)
    print('Total %d found in %dms' % (results['hits']['total'], results['took']))
    print('-' * 80)

def search_query(query):
    """Executes a call to elastic via q param"""
    print_hits(es.search(index='shakespeare', params={"q": query}))


def search_query_body(body):
    """Executes a call to elastic via q param"""
    print_hits(es.search(index='shakespeare', body=body))
```

## Busquedas

### Buscar en el campo `_all`

Elastic permite hacer una busqueda en todos los campos a la vez, gracias a  
un campo especial llamado `_all` disponible en las queries:


```python
search_query("love")
```

    ================================================================================
    Total 1857 found in 173ms
    --------------------------------------------------------------------------------
    /shakespeare/doc/100798 [6.4629364]: |Troilus and Cressida| PARIS - Ay, good now, love, love, nothing but love.
    /shakespeare/doc/100801 [6.321299]: |Troilus and Cressida| PANDARUS - Love, love, nothing but love, still more!
    /shakespeare/doc/36481 [5.636754]: |Hamlet| LAERTES - I do receive your offerd love like love,
    /shakespeare/doc/53622 [5.636754]: |Loves Labours Lost| MOTH - swallowed love with singing love, sometime through
    /shakespeare/doc/75906 [5.540254]: |Pericles| PERICLES - Few love to hear the sins they love to act;
    /shakespeare/doc/34585 [5.5334353]: |Hamlet| Player King - Whether love lead fortune, or else fortune love.
    /shakespeare/doc/45125 [5.5334353]: |King John| ARTHUR - Nay, you may think my love was crafty love
    /shakespeare/doc/74070 [5.5334353]: |Othello| IAGO - Ill love no friend, sith love breeds such offence.
    /shakespeare/doc/35965 [5.5285854]: |Hamlet| First Clown - In youth, when I did love, did love,
    /shakespeare/doc/81720 [5.5285854]: |Richard III| GLOUCESTER - Shall, for thy love, kill a far truer love;
    ================================================================================


### Busqueda por campos

Tambien podemos hacer busquedas directas en cada campo:


```python
search_query("speaker:(ROMEO OR JULIET)")
```


```python
search_query("speaker:(ROMEO OR JULIET) AND NOT speaker:JULIET")
```

### Busqueda con boosts

Tambien podemos hacer busquedas con ciertos campos potenciados:


```python
search_query("text_entry:love AND (speaker:ROMEO^5 OR speaker:JULIET)")
```

### Busqueda con comodines o wildcards

Tambien podemos hacer busquedas con comodines dentro de campos:


```python
search_query("play_name:ot?ello")
```


```python
search_query("play_name:K* AND NOT play_name:*John*") #Remove John
```

    ================================================================================
    Total 3766 found in 197ms
    --------------------------------------------------------------------------------
    /shakespeare/doc/49031 [1.0]: |King Lear| KENT - Albany than Cornwall.
    /shakespeare/doc/49038 [1.0]: |King Lear| GLOUCESTER - His breeding, sir, hath been at my charge: I have
    /shakespeare/doc/49043 [1.0]: |King Lear| GLOUCESTER - she grew round-wombed, and had, indeed, sir, a son
    /shakespeare/doc/49047 [1.0]: |King Lear| KENT - being so proper.
    /shakespeare/doc/49052 [1.0]: |King Lear| GLOUCESTER - fair; there was good sport at his making, and the
    /shakespeare/doc/49053 [1.0]: |King Lear| GLOUCESTER - whoreson must be acknowledged. Do you know this
    /shakespeare/doc/49057 [1.0]: |King Lear| GLOUCESTER - honourable friend.
    /shakespeare/doc/49058 [1.0]: |King Lear| EDMUND - My services to your lordship.
    /shakespeare/doc/49064 [1.0]: |King Lear| KING LEAR - Attend the lords of France and Burgundy, Gloucester.
    /shakespeare/doc/49065 [1.0]: |King Lear| GLOUCESTER - I shall, my liege.
    ================================================================================



```python
search_query("text_entry:kil? AND speaker:(*king*^4)")
```

### Busqueda con Fuzziness

Tambien podemos hacer busquedas con fuzzyness:


```python
search_query("text_entry:inocent~1")
```


```python
search_query_body({
  "query": {
    "query_string": {
      "query": "love AND (NOT play_name:\"Romeo and Juliet\")"
    }
  }
})
```

    ================================================================================
    Total 1730 found in 139ms
    --------------------------------------------------------------------------------
    /shakespeare/doc/100798 [7.4629364]: |Troilus and Cressida| PARIS - Ay, good now, love, love, nothing but love.
    /shakespeare/doc/100801 [7.321299]: |Troilus and Cressida| PANDARUS - Love, love, nothing but love, still more!
    /shakespeare/doc/36481 [6.636754]: |Hamlet| LAERTES - I do receive your offerd love like love,
    /shakespeare/doc/53622 [6.636754]: |Loves Labours Lost| MOTH - swallowed love with singing love, sometime through
    /shakespeare/doc/75906 [6.540254]: |Pericles| PERICLES - Few love to hear the sins they love to act;
    /shakespeare/doc/34585 [6.5334353]: |Hamlet| Player King - Whether love lead fortune, or else fortune love.
    /shakespeare/doc/45125 [6.5334353]: |King John| ARTHUR - Nay, you may think my love was crafty love
    /shakespeare/doc/74070 [6.5334353]: |Othello| IAGO - Ill love no friend, sith love breeds such offence.
    /shakespeare/doc/35965 [6.5285854]: |Hamlet| First Clown - In youth, when I did love, did love,
    /shakespeare/doc/81720 [6.5285854]: |Richard III| GLOUCESTER - Shall, for thy love, kill a far truer love;
    ================================================================================


## Indexes y maps

Antes de insertar datos es recomendable siempre crear el indice y sus mapeos:


```python
es.indices.create(index='documents_march', body={
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 3,
    "analysis": {},
    "refresh_interval": "1s"
  },
  "mappings": {
    "document": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "english"
        },
        "category": {
            "type": "keyword"
        },
        "description": {
            "type":"text",
            "analyzer": "english"
        }
      }
    }
  }
})
# DEPRECATED! 1 type in elastic > 6.0.0
# Fine tunning with the new keywords category and description


```




    {'acknowledged': True, 'shards_acknowledged': True, 'index': 'documents_march'}



Esto es equivalente a:
```
PUT /documents
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 3,
    "analysis": {},
    "refresh_interval": "1s"
  },
  "mappings": {
    "title_text": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "english"
        }
      }
    }
  }
}
```


```python
es.indices.get("documents_february")
# Equivalente a GET /documents_february [_settings|_mappings]
```




    {'documents_february': {'aliases': {},
      'mappings': {'document': {'properties': {'category': {'type': 'keyword'},
         'description': {'type': 'text', 'analyzer': 'english'},
         'title': {'type': 'text', 'analyzer': 'english'}}}},
      'settings': {'index': {'refresh_interval': '1s',
        'number_of_shards': '3',
        'provided_name': 'documents_february',
        'creation_date': '1558469830151',
        'number_of_replicas': '1',
        'uuid': 'iVzmmf6YTXyytCwQXnnm6Q',
        'version': {'created': '5060499'}}}}}



### Modificar tipos


```python
es.indices.put_mapping(index='documents_february', doc_type='document', body={
  "document": {
    "properties": {
      "content": {
          "type": "text",
          "analyzer": "english"
      },
    }
  }
})
es.indices.put_mapping(index='documents_february', doc_type='document', body={
  "document": {
    "properties": {
      "tag": {
          "type": "keyword"
      },
    }
  }
})
es.indices.get("documents_february")
```




    {'documents_february': {'aliases': {},
      'mappings': {'document': {'properties': {'category': {'type': 'keyword'},
         'content': {'type': 'text', 'analyzer': 'english'},
         'description': {'type': 'text', 'analyzer': 'english'},
         'tag': {'type': 'keyword'},
         'title': {'type': 'text', 'analyzer': 'english'}}}},
      'settings': {'index': {'refresh_interval': '1s',
        'number_of_shards': '3',
        'provided_name': 'documents_february',
        'creation_date': '1558469830151',
        'number_of_replicas': '1',
        'uuid': 'iVzmmf6YTXyytCwQXnnm6Q',
        'version': {'created': '5060499'}}}}}



Esto es equivalente a:
    ```
    PUT /documents_february/_mapping/document
    {
      "document": {
        "properties": {
          "tag": {
            "type": "keyword"
          }
        }
      }
    }
    ```
    
## Manejo de documentos

Hagamos CRUD sobre documentos:


```python
es.create(index='documents_february', doc_type='document', id=0, body={
    "title": "New Document",
    "content": "This is a new document for the master class",
    "tag": ["general", "testing"]
})


```




    {'_index': 'documents_february',
     '_type': 'document',
     '_id': '0',
     '_version': 1,
     'result': 'created',
     '_shards': {'total': 2, 'successful': 1, 'failed': 0},
     'created': True}



Equivalente a:
```
PUT /documents_february/document/1
{
  "title": "New Document",
  "content": "This is a new document for the master class",
  "tag": [
    "testing"
  ]
}
-----

POST /documents_february/document
{
  "title": "New Document",
  "content": "This is a new document for the master class",
  "tag": [
    "testing"
  ]
}
```

### Lectura


```python
es.get(index='documents_february', doc_type='document', id=0)



```




    {'_index': 'documents_february',
     '_type': 'document',
     '_id': '0',
     '_version': 1,
     'found': True,
     '_source': {'title': 'New Document',
      'content': 'This is a new document for the master class',
      'tag': ['general', 'testing']}}



Equivalente a:
```
GET /documents_february/document/0
```

### Borrado


```python
es.delete(index='documents_february', doc_type='document', id=0)


```




    {'found': True,
     '_index': 'documents_february',
     '_type': 'document',
     '_id': '0',
     '_version': 2,
     'result': 'deleted',
     '_shards': {'total': 2, 'successful': 1, 'failed': 0}}



Equivalente a:
```
DELETE /documents_february/document/0
```

## Agregacion y Queries complejas

Elastic permite, gracias a su API de agregacion, realizar queries mas complejas:

```
POST /_search

{
    "size":0,
    "aggs" : {
        "Popular plays" : {
            "terms" : {
                "field" : "play_name.keyword"
            }
        }
    }
}

-------
{
    "size":0,
    "aggs" : {
        "Total plays" : {
            "terms" : {
                "field" : "play_name.keyword"
            },
            "aggs" : {
             "Per type" : {
                 "terms" : {
                     "field" : "speaker.keyword"
                  }
             }
            }
        }
    }
}

-------
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "speaker": "*king*"
          }
        }
      ]
    }
  },
  "aggs": {
    "my_agg": {
      "terms": {
        "field": "play_name.keyword",
        "size": 10
      }
    }
  },
  "sort": [
    {
      "play_name.keyword": {
        "order": "desc"
      }
    }
  ]
}
```
