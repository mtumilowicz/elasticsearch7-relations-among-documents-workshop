[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

# elasticsearch7-relations-among-documents-workshop

* reference
    * https://www.factweavers.com/blog/join-in-elasticsearch/
    * https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-request-body.html#request-body-search-inner-hits
    * https://www.waitingforcode.com/elasticsearch/reverse-nested-aggregation-in-elasticsearch/read
    * https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html
    * https://blog.mimacom.com/parent-child-elasticsearch/
    * https://www.manning.com/books/elasticsearch-in-action

## preface
* goals of this workshop
    * https://github.com/mtumilowicz/java12-elasticsearch-inverted-index-workshop
    * https://github.com/mtumilowicz/elasticsearch7-query-filter-aggregation-workshop
    * introduction to inner documents
    * defining relations between documents
    * show difference between types od relations: object, nested and join
    * simple examples of queries and aggregations for inner documents
* in `docker-compose` there is elasticsearch + kibana (7.6) prepared for local testing
    * cd `docker/compose`
    * `docker-compose up -d`
* workshop and answers are in `workshop` directory

## object
* objects work best for one-to-one relationships
* everything is in the same document - best performance
* no joins
* at the Lucene level they are flat documents
* main drawback: updating a single object will re-index the whole document
    * every update is actually a delete + reindex operation.
* in case of arrays - document is not aware of boundaries between objects

### example
* single
    * `mappings.properties`
        ```
        "location": { 
            "properties": {
                "name": { "type": "text" },
                "geolocation": { "type": "geo_point" }
            }
        }
        ```
    * original document
        ```
        {
            "location": {
                "name": "Warsaw",
                "geolocation": {
                    "lat": 52.22977,
                    "lon": 21.01178
                }
            }
        }
        ```
    * document stored in Lucene
        ```
        {
            "location.name": "Warsaw",
            "location.geolocation.lat": 52.22977
            "location.geolocation.lon": 21.01178
        }
        ```
    * query
        ```
        "query" : { "match": { "location.name": "Warsaw" } }
        ```
* array
    * `mappings.properties`
        ```
        "location": { 
            "properties": {
                "name": { "type": "text" },
                "geolocation": { "type": "geo_point" }
            }
        }
        ```
    * original document
        ```
        {
            "location": [
                {
                    "name": "Warsaw",
                    "geolocation": {
                        "lat": 52.22977,
                        "lon": 21.01178
                },
                {
                    "name": "Cracow",
                    "geolocation": {
                        "lat": 50.06143,
                        "lon": 19.93658
                }
            ]
        }
        ```
    * document stored in Lucene
        ```
        {
            "location.name": ["Warsaw", "Cracow"]
            "location.geolocation.lat": [52.22977, 50.06143]
            "location.geolocation.lon": [21.01178, 19.93658]
        }
        ```
    * example: lost boundary
        * query: 
            ```
            "must": [
                 { "match": { "location.name": "Warsaw" } },
                 { "geo_distance": {
                     "distance": "10km",
                     "location.geolocation": {
                       "lat": 50.06143, // Cracow.lat
                       "lon": 19.93658 // Cracow.lon
                     }
                   }
                 }
               ]
            ```
        * result
            ```
            "hits": [
              {
                "_source": {
                  "location": [
                    {
                      "name": "Warsaw",
                      "geolocation": {
                        "lat": 52.22977,
                        "lon": 21.01178
                      }
                    },
                    {
                      "name": "Cracow",
                      "geolocation": {
                        "lat": 50.06143,
                        "lon": 19.93658
                      }
                    }
                  ]
                }
              }
            ]
            ```
            * explication
                * location.name = Warsaw? => yes
                * any geolocation near Cracow => yes

## nested
* Elasticsearch internally splits each nested object into its own Lucene document
    * stored in a single block (adjacent on disk)
* they get fetched and queried with the minimum number of operations
    * objects are grouped into a contiguous block with the parent
* Lucene sees them as separate docs
    * Elasticsearch logically treats them as one logical document
        * logical block = contiguous group of Lucene documents 
        * in particular: nested queries and filters will join them
    * Lucene doesn’t store explicit parent-child links
        * Elasticsearch uses document order within a block to infer grouping
* main drawback: updating or adding one inner document requires re-indexing the whole block
    * update = all nested documents and the parent must be deleted and re-indexed together
    * reason: only way to preserve the logical structure is to rebuild the whole block
        * there is no way to say "Recreate Child3 (updated nested)"
* some extra work to join multiple documents within a block
    * much faster than joining documents that not reside in same blocks

### example
* single
    * `mappings.properties`
        ```
        "location": {
            "type": "nested",
            "properties": {
                "name": { "type": "text" },
                "geolocation": { "type": "geo_point" }
            }
        }
        ```
    * query
        * `path` - path to the nested object you wish to search
        ```
        "nested": {
            "path": "location",
            "query" : { "match": { "location.name": "Warsaw" } }
        }
        ```
    * query with inner hits
        * `inner_hits` - additional nested hits that caused a search hit to match
            * in many cases, it’s very useful to know which inner nested objects caused
            certain information to be returned
        ```
        "nested": {
            "path": "location",
            "query" : { "match": { "location.name": "Warsaw" } },
            "inner_hits": {}
        }
        ```
* array
    * `mappings.properties`
        ```
        "location": {
            "type": "nested",
            "properties": {
                "name": { "type": "text" },
                "geolocation": { "type": "geo_point" }
            }
        }
        ```
    * query
        ```
        "nested": {
            "path": "location",
            "query" : { "match": { "location.name": "Warsaw" } }
        }
        ```
* aggregations
    * examples
        * count employees by title
            ```
            // GET /company_index/_search
          
            "aggs": {
              "employees_nested": {
                "nested": {
                  "path": "employees" // goes into the employees nested field (nested)
                },
                "aggs": {
                  "by_title": {
                    "terms": {
                      "field": "employees.title" // performs a terms aggregation on employees.title
            ```
        * find all titles, and for each title, count how many companies have employees with that title
            ```
            // GET /company_index/_search
          
            "aggs": {
              "employees_nested": {
                "nested": {
                  "path": "employees"
                },
                "aggs": {
                  "by_title": {
                    "terms": {
                      "field": "employees.title"
                    },
                    "aggs": {
                      "companies_with_this_title": {
                        "reverse_nested": {}  
            ```

## join
* creates parent/child relation within documents of the same index
* ability to modify a child object independent of the parent
* comparing to nested
    * searches are slower
    * indexing, updating, and deleting is faster
* useful when many connected documents have to be indexed asynchronously
* children and parent have to be in the same shard
    * indexing - explicit routed using parent id

### example
* single
    1. `mappings.properties`
        ```
        "parent": { "type": "text" },
        "child": { "type": "text" },
        "jukebox_relations": {
            "type": "join",
            "relations": {
                "parent": "child"
            }
        }
        ```
    1. indexing
        * parent
            ```
            POST index-name/_doc/id
            {
                ...,
                "jukebox_relations": "parent"
            }
            ```
        * child
            ```
            POST index-name/_doc/childId?routing=parentId
            {
                ...,
                "jukebox_relations": { "name": "child", "parent": parentId }
            }
            ```
    1. query
        * `has_child`
            * returns parent documents whose joined child documents match a provided query
        * `has_parent`
            * returns child documents whose joined parent document matches a provided query
        ```
        "has_parent": {
            "parent_type": "parent",
            "query": { "match": { "name": "Michal" } }
        }
        ```
* aggregations
    * example: count employees by title
        ```
        // GET /company_index/_search
      
        "aggs": {
          "employees": {
            "children": {
              "type": "employee" // selects all documents of type employee; refers to the join name
            },
            "aggs": {
              "titles": {
                "terms": {
                  "field": "title"  
        ```