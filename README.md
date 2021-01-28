# Parte 2 del práctico de elasticsearch para la optativa de Recuperación de información, de la Diplomatura en Ciencias de Datos, FaMAF.

# ej 1

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

devuelve 587 resultados vs 0 resultados de query "PRESIDENT"

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




