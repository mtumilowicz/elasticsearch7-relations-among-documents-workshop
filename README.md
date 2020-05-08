* https://www.factweavers.com/blog/join-in-elasticsearch/
* https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-request-body.html#request-body-search-inner-hits
* https://www.waitingforcode.com/elasticsearch/reverse-nested-aggregation-in-elasticsearch/read
* https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html
* https://blog.mimacom.com/parent-child-elasticsearch/

## object
* objects work best for one-to-one relationships
* everything is in the same document - best performance
* no joins
* at the Lucene level they are flat documents
    * running queries and aggregations on objects - dot notation
* main drawback: updating a single object will re-index the whole document

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
        { "match": { "object.field": "value" } }
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
        * note that document is not aware of boundaries between objects

## nested
* at the Lucene level, Elasticsearch will index the root document and all the members 
objects in separate documents
    * it will put them in a single block
* Documents of a block will always stay together, ensuring they get fetched and queried
  with the minimum number of operations.
* specify nested mapping to get them indexed as separate documents in the same block
* nested queries and filters will trigger Elasticsearch to join the different Lucene documents within
  the same block and treat the resulting data as single document
* some extra work to join multiple documents within a block
    *  much faster than joining documents that not reside in same blocks
* updating or adding one inner document requires re-indexing the whole block

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
        ```
        "nested": {
            "path": "location",
            "query" : { "match": { "location.name": "Warsaw" } }
        }
        ```
    * query with inner hits
        * inner hits - additional nested hits that caused a search hit to match
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
* note that we could mix nested and objects
    * `copy_to` - allows you to copy the values of multiple fields into a group field, which can then 
    be queried as a single field
        * original _source field will not be modified to show the copied values
    * `mappings.properties`
        ```
        "locationNames": { "type": "text" },
        "location": {
            "type": "nested",
            "properties": {
                "name": { 
                    "type": "text",
                    "copy_to": "locationNames"      
                },
                "geolocation": { "type": "geo_point" }
            }
        }
        ```
* aggregations
    * `reverse_nested`
        * single bucket aggregation that enables aggregating on parent docs from nested documents
        * must be defined inside a nested aggregation
        * it joins back to the root / main document level
            * it is configurable - `path`
            
## join
* creates parent/child relation within documents of the same index
* ability to modify a child object independent of the parent

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
    * `children`
        * special single bucket aggregation that selects child documents that have the specified type