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

> En este ejercicio nos piden buscar por el campo place.bounding_box, usando coordenadas.

No todos los registros del dataset tienen este campo, así que primero hay que filtrar aquellos que si lo tienen - 154 de los 9567 registros tienen este campo, como lo evidencia la query correspondiente:

```javascript
GET g20/_search
{"_source":["place.bounding_box.coordinates"],
 "query": {
    "bool": {
      "filter": {
        "exists": {
          "field": "place.bounding_box.coordinates"
        }
      }}}}

...

"hits" : {
    "total" : {
      "value" : 154,
      "relation" : "eq"
    },

```


y uno de estos hits se ve da la siguiente forma:

```javascript
{
        "_index" : "g20",
        "_type" : "_doc",
        "_id" : "1059862707782987777",
        "_score" : 0.0,
        "_source" : {
          "place" : {
            "bounding_box" : {
              "coordinates" : [
                [
                  [
                    72.74484,
                    18.845343
                  ],
                  [
                    72.74484,
                    19.502937
                  ],
                  [
                    73.003648,
                    19.502937
                  ],
                  [
                    73.003648,
                    18.845343
                  ]
                ]
              ]
            }
          }
        }
      }
```
que tiene sentido, serían los 4 puntos de la caja que delimita la zona geográfica.
Esto nos refiere a un tipo de dato específico que elasticsearch soporta: el [Geo bounding box](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-query.html), al cual se le pueden hacer queries complejas igual que al tipo de dato Date.
Vemos que por defecto, este campo fue indexado por elasticsearch como float:

```javascript
GET /g20/_mapping/field/place.bounding_box.coordinates

{
  "g20" : {
    "mappings" : {
      "place.bounding_box.coordinates" : {
        "full_name" : "place.bounding_box.coordinates",
        "mapping" : {
          "coordinates" : {
            "type" : "float"
          }
        }
      }
    }
  }
}
```

Por lo que habría que establecer un mapping a geo_shape, ya que es un quadtree: cuatro geo_points que delimitan una shape geométrica.

Se lo indexa haciendo:


El coerce se setteó a true porque sino al cargar los datos daba el siguiente error:
```javascript
"error": {
                    "type": "mapper_parsing_exception",
                    "reason": "failed to parse field [place.bounding_box] of type [geo_shape]",
                    "caused_by": {
                        "type": "x_content_parse_exception",
                        "reason": "Failed to build [geojson] after last required field arrived",
                        "caused_by": {
                            "type": "illegal_argument_exception",
                            "reason": "first and last points of the linear ring must be the same (it must close itself): x[0]=72.74484 x[3]=73.003648 y[0]=18.845343 y[3]=18.845343"
                        }
                    }
```
> Pareciera ser que hay errores en los datos originales.

ahora al buscar 
```javascript
GET ej_5/_search
{"_source":["place.bounding_box.coordinates"],
 "query": {
    "bool": {
      "filter": {
        "exists": {
          "field": "place.bounding_box"
        }
      }}}}
```
obtenemos los 149 tweets que tienen esa información, que aparecen en el mapping original.

Una vez indexado podemos hacer este tipo de queries:
```javascript
GET ej_5/_search
{"query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_shape": {
          "place.bounding_box": {
            "shape": {
              "type": "polygon",
              "coordinates": [[[-122.6735372003,40.1846173338],[-103.1789051946,40.1846173338],[-103.1789051946,48.2617420987],[-122.6735372003,48.2617420987],[-122.6735372003,40.1846173338]]]
            },
            "relation": "within"
          }
        }
      }
    }
  }
}
```
ie buscar aquellos tweets que fueron tweeteados dentro de  esta región en el mapa [fuente](https://boundingbox.klokantech.com/):
<img src="https://i.imgur.com/RlkiUwb.png">


Nos da 16 resultados, por ej:

```javascript
      {
        "_index" : "ej_5",
        "_type" : "_doc",
        "_id" : "1059863923183124481",
        "_score" : 1.0,
        "_source" : {
          "quote_count" : 0,
          "contributors" : null,
          "truncated" : false,
          "text" : "@realDonaldTrump If you're against the Mexican Caravan, you are for the Mexican Drug Cartels",
          "is_quote_status" : false,
          "in_reply_to_status_id" : 1059469158830796800,
          "reply_count" : 0,
          "id" : 1059863923183124481,
          "favorite_count" : 0,
          "entities" : {
            "user_mentions" : [
              {
                "id" : 25073877,
                "indices" : [
                  0,
                  16
                ],
                "id_str" : "25073877",
                "screen_name" : "realDonaldTrump",
                "name" : "Donald J. Trump"
              }
            ],
            "symbols" : [ ],
            "hashtags" : [ ],
            "urls" : [ ]
          },
          "retweeted" : false,
          "coordinates" : null,
          "timestamp_ms" : "1541526225263",
          "source" : """<a href="http://twitter.com/download/android" rel="nofollow">Twitter for Android</a>""",
          "in_reply_to_screen_name" : "realDonaldTrump",
          "id_str" : "1059863923183124481",
          "display_text_range" : [
            17,
            92
          ],
          "retweet_count" : 0,
          "in_reply_to_user_id" : 25073877,
          "favorited" : false,
          "user" : {
            "follow_request_sent" : null,
            "profile_use_background_image" : true,
            "default_profile_image" : false,
            "id" : 55642438,
            "default_profile" : false,
            "verified" : false,
            "profile_image_url_https" : "https://pbs.twimg.com/profile_images/831171445560455168/SW2pNTrE_normal.jpg",
            "profile_sidebar_fill_color" : "252429",
            "profile_text_color" : "666666",
            "followers_count" : 527,
            "profile_sidebar_border_color" : "181A1E",
            "id_str" : "55642438",
            "profile_background_color" : "1A1B1F",
            "listed_count" : 4,
            "profile_background_image_url_https" : "https://abs.twimg.com/images/themes/theme9/bg.gif",
            "utc_offset" : null,
            "statuses_count" : 6169,
            "description" : "It's scary how people are fooled by inconsistent propaganda.  Let's Educate and #resist the insanity. #Resistance",
            "friends_count" : 1447,
            "location" : "Earth",
            "profile_link_color" : "2FC2EF",
            "profile_image_url" : "http://pbs.twimg.com/profile_images/831171445560455168/SW2pNTrE_normal.jpg",
            "following" : null,
            "geo_enabled" : true,
            "profile_banner_url" : "https://pbs.twimg.com/profile_banners/55642438/1519508860",
            "profile_background_image_url" : "http://abs.twimg.com/images/themes/theme9/bg.gif",
            "name" : "Jason Nebenfuhr",
            "lang" : "en",
            "profile_background_tile" : false,
            "favourites_count" : 12332,
            "screen_name" : "the_Neb",
            "notifications" : null,
            "url" : null,
            "created_at" : "Fri Jul 10 19:26:18 +0000 2009",
            "contributors_enabled" : false,
            "time_zone" : null,
            "protected" : false,
            "translator_type" : "none",
            "is_translator" : false
          },
          "geo" : null,
          "in_reply_to_user_id_str" : "25073877",
          "lang" : "en",
          "created_at" : "Tue Nov 06 17:43:45 +0000 2018",
          "filter_level" : "low",
          "in_reply_to_status_id_str" : "1059469158830796800",
          "place" : {
            "country_code" : "US",
            "url" : "https://api.twitter.com/1.1/geo/id/8d71376556a9e531.json",
            "country" : "United States",
            "place_type" : "city",
            "bounding_box" : {
              "type" : "Polygon",
              "coordinates" : [
                [
                  [
                    -122.309297,
                    47.343399
                  ],
                  [
                    -122.309297,
                    47.441224
                  ],
                  [
                    -122.126854,
                    47.441224
                  ],
                  [
                    -122.126854,
                    47.343399
                  ]
                ]
              ]
            },
            "full_name" : "Kent, WA",
            "attributes" : { },
            "id" : "8d71376556a9e531",
            "name" : "Kent"
          }
        }
      },
```
