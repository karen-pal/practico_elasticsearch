# Parte 2 del práctico de elasticsearch para la optativa de Recuperación de información, de la Diplomatura en Ciencias de Datos, FaMAF.

# ej 1

> en este ejercicio se pide investigar cómo poder buscar en elasticsearch de forma case-sensitive, de tal forma que la búsqueda de "PRESIDENT" de distintos resultados que "president".

Por defecto elasticsearch indexa los campos usando el standard analyzer, que al indexar pasa las palabras a lowercase y por consecuencia hace la búsqueda case-insensitive

Hay que crear, luego, un nuevo mapping para este campo "text", lo que se hace de la siguiente forma:

```javascript
PUT ej_1
{
  "settings": {
    "analysis": {
      "analyzer": {
        "case_sensitive": {
          "tokenizer": "whitespace",
          "type":"custom"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "text":{"type": "text",
          "analyzer": "case_sensitive"}
      }
  }
}

```
Luego de hacer el bulk upload de la info de g20 podemos ver que la query:

```javascript
GET /ej_1/_search
{"_source":["text"],
  "query":
    { "match":
      { "text":
        {"query":"President"}
      }
    }
}
```

devuelve 441 resultados. Mientras que la query
```javascript
GET /ej_1/_search
{"_source":["text"],
  "query":
    { "match":
      { "text":
        {"query":"president"}
      }
    }
}
```
devuelve 20. Y si quereeamos con "PRESIDENT" obtenemos 4.
> Esto se contrasta con los 587 resultados que obteníamos antes, cuando -por las configuraciones por defecto de elasticsearch- no teníamos la posibilidad de buscar case sensitive-ly.

# ej 2

> En este ejercicio se pide poder buscar por palabras que contengan una misma raíz. Para que por ejemplo podamos construir una query tal que al buscar por "pray" nos aparezcan resultados como "praying", "prayed", "prays", etc.

Esto nos refiere al stemming que tiene en cuenta elasticsearch dentro de sus token filters, [como nos indica la documentación](https://www.elastic.co/guide/en/elasticsearch/reference/current/stemming.html).

Para esto no parece que pudiéramos especificar el tokenizer en un query específico, sino que hay que hacer un analyzer custom.
[Por ej acá en stack overflow](https://stackoverflow.com/questions/32103886/match-query-internals-on-stemmed-word-search-elastic)

Una vez creado, hay que tener cuidado con cómo se busca. Se debe buscar con una match query para que primero se busque en el output que se obtiene usando el analizador custom, [explicado con ejemplos acá](https://stackoverflow.com/questions/32103886/match-query-internals-on-stemmed-word-search-elastic)


```javascript
PUT ej_2
{
   "settings": {
      "analysis": {
         "analyzer": {
            "stemmer_analyzer": {
                "type":"custom",
               "tokenizer": "whitespace",
               "filter": [
                  "my_stemmer"
               ]
            }
         },
         "filter": {
            "my_stemmer": {
               "type": "stemmer",
               "name": "english"
            }
         }
      }
   }
}

PUT ej_2/_mapping
{
     "properties": {
        "text": {
           "type": "text",
           "analyzer": "stemmer_analyzer"
        }
     }
}
```
Luego la query

```javascript
GET /ej_2/_search
{"_source":["text"],
  "query": {
    "bool": {
      "must":
        { "match": { "text": "pray" } }
    }
  }
}
```
Nos da el resultado pedido. Esto se puede ver más en detalle comparando los outputs de
```javascript
GET /ej_2/_analyze
{
  "field": "text",
  "text": "praying prays pray"
}
```

vs la misma query en otra colección indexada de forma standard (sin stemming)

```javascript
GET /g20/_analyze
{
  "field": "text",
  "text": "praying prays pray"
}
```

vemos los tokens que generó el primero son
```javascript
{
  "tokens" : [
    {
      "token" : "prai",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "prai",
      "start_offset" : 8,
      "end_offset" : 13,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "prai",
      "start_offset" : 14,
      "end_offset" : 18,
      "type" : "word",
      "position" : 2
    }
  ]
}
```

vs

```javascript
{
  "tokens" : [
    {
      "token" : "praying",
      "start_offset" : 0,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "prays",
      "start_offset" : 8,
      "end_offset" : 13,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "pray",
      "start_offset" : 14,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```

# ej 3
> En este ejercicio se pide una forma de buscar matches exactos en campos de tipo texto, específicamente en el campo user.location

Vemos que estas siguientes queries nos traen resultados no deseados:

```javascript
GET /g20/_search
{"_source":["user.location"],
  "query": { 
    "constant_score" : { 
            "filter" : {
                "term" : { 
                    "user.location": "york"
                }
            }
        }
    }
}


GET /g20/_search
{"_source":["user.location"],
  "query": { 
    "term": {
      "user.location": {
        "value": "york"
      }
    }
  }
}

```
> 103 resultados, entre ellos "new york new york", etc. pasa lo mismo si reemplazamos "york" con "california", y obtenemos 147 resultados.


En cambio con estas queries:
```javascript
GET /g20/_search
{"_source":["user.location"],
  "query": { "match_phrase": { "user.location.keyword": "California" } }
}


GET /g20/_search
{"_source":["user.location"],
  "query": {
    "term": {
      "user.location.keyword": {
        "value": "California"
      }
    }
  }
}
```

obtenemos 5 valores, que constituyen matches exactos (incluso en capitalización).
Esto se refiere al hecho de que elasticsearch al indexar campos de tipo texto, crea un subcampo "keyword" de forma automática (de tipo keyword), lo que nos deja hacer búsquedas exactas en term queries accediendo de la forma ```<field_name>.keyword```.


# ej 4
> En este ejercicio se pide consultar un campo por rangos de valores.

Este campo es una fecha, pero por defecto elasticsearch lo indexó como tipo texto

```javascript
GET /g20/_search
{"_source":["user.created_at"],
    "query": { "match_all": {} }

}
```
un ejemplo de un hit
```javascript
{
        "_index" : "g20",
        "_type" : "_doc",
        "_id" : "1059862694315069440",
        "_score" : 1.0,
        "_source" : {
          "user" : {
            "created_at" : "Sun Aug 14 18:38:13 +0000 2011"
          }
        }
      }
```

podemos inspeccionar los mappings y tipos de este campo con el comando

```javascript
GET g20/_mapping/field/user.created_at


{
  "g20" : {
    "mappings" : {
      "user.created_at" : {
        "full_name" : "user.created_at",
        "mapping" : {
          "created_at" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          }
        }
      }
    }
  }
}
```
para poder explotar las posibilidades de agregaciones con fechas hay que remappear este campo para que sea de [tipo date](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html).


Vemos que [tiene el formato](https://stackoverflow.com/questions/63536502/elasticsearch-failed-to-parse-date-field)

> Sun Aug 14 18:38:13 +0000 2011
> EEE MMM dd HH:mm:ss Z yyyy

por lo que defino el mapping

```javascript
PUT ej_4
{
  "mappings": {
    "properties": {
      "user.created_at": {
        "type":   "date",
        "format": "EEE MMM dd HH:mm:ss Z yyyy"
      }
    }
  }
}
```

Luego puedo hacer queries de la forma:

```javascript
GET /ej_4/_search
{"_source": "user.created_at", 
 "query": { 
    "bool": { 
      "must": { "match_all": {} }, 
      "filter": { 
        "range": { 
          "user.created_at": { 
            "gte": "Sun Aug 14 18:38:13 +0000 2011"
            } } } } } }
```

es decir, filtrar por los usuarios que ingresaron el día o después del día 14/8/2011, y obtengo 7231 resultados, por ej:
```javascript
{
        "_index" : "ej_4",
        "_type" : "_doc",
        "_id" : "1059862698010271744",
        "_score" : 1.0,
        "_source" : {
          "user" : {
            "created_at" : "Sun Nov 29 08:33:40 +0000 2015"
          }
        }
      }
```

mientras que moviendo la fecha una semana (21/8/2011) obtengo 7200 resultados, por ej:

```javascript
{
        "_index" : "ej_4",
        "_type" : "_doc",
        "_id" : "1059862694931693571",
        "_score" : 1.0,
        "_source" : {
          "user" : {
            "created_at" : "Wed Dec 10 08:30:08 +0000 2014"
          }
        }
      }
```


# ej 5
