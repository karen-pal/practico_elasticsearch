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
```javascrip
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
[Por ej acá en stack overflow](https://stackoverflow.com/questions/32229255/elasticsearch-match-with-stemming)

Una vez creado, hay que tener cuidado con cómo se busca. Se debe buscar con una match query para que primero se busque en el output que se obtiene usando el analizador custom, [explicado con ejemplos acá](https://stackoverflow.com/questions/32103886/match-query-internals-on-stemmed-word-search-elastic)





# ej 3
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
103 resultados, entre ellos "new york new york", etc. pasa lo mismo si reemplazamos "york" con "california", y obtenemos 147 resultados.


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
