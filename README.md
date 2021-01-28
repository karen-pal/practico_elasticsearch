# Parte 2 del práctico de elasticsearch para la optativa de Recuperación de información, de la Diplomatura en Ciencias de Datos, FaMAF.

# ej 1

> en este ejercicio se pide investigar cómo poder buscar en elasticsearch de forma case-sensitive, de tal forma que la búsqueda de "PRESIDENT" de distintos resultados que "president".

> La resolución a esto es la siguiente query:

```javascript
GET /g20/_search
{
  "query": 
    { "match": 
      { "text": 
        {"query":"president",
        "analyzer": "whitespace"
        }
      }
    }
}
```

devuelve 587 resultados vs 0 resultados de esta misma query con query.match.text.query = "PRESIDENT"

en comparación

```javascript
GET /g20/_search
{
  "query": {
    "bool": {
      "must":
        { "match": { "text": "PRESIDENT" } }
    }
  }
}
```
devuelve los 587 resultados.
Esto tiene que ver con el analyzer por defecto de elasticsearch es el standard analyzer, que al indexar para hacer la búsqueda pasa todo a minúsculas, eliminando la posibilidad de búsquedas case-sensitive

Esto también se puede resolver cambiando qué search analyzer se usa por defecto para este campo, [como dice la documentación](https://www.elastic.co/guide/en/elasticsearch/reference/current/specify-analyzer.html#specify-search-query-analyzer)

# ej 2

> En este ejercicio se pide poder buscar por palabras que contengan una misma raíz. Para que por ejemplo podamos construir una query tal que al buscar por "pray" nos aparezcan resultados como "praying", "prayed", "prays", etc.

Esto nos refiere al stemming que tiene en cuenta elasticsearch dentro de sus token filters, [como nos indica la documentación](https://www.elastic.co/guide/en/elasticsearch/reference/current/stemming.html).

Para esto no parece que pudiéramos especificar el tokenizer en un query específico, sino que hay que hacer un analyzer custom.
[Por ej acá en stack overflow](https://stackoverflow.com/questions/32229255/elasticsearch-match-with-stemming)

Una vez creado, hay que tener cuidado con cómo se busca. Se debe buscar con una match query para que primero se busque en el output que se obtiene usando el analizador custom, [explicado con ejemplos acá](https://stackoverflow.com/questions/32103886/match-query-internals-on-stemmed-word-search-elastic)
