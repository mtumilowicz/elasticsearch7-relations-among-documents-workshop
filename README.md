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
    
## join
* The join datatype is a special field that creates parent/child relation within documents of the same index

PUT jukebox
{
    "mappings": {
        "properties": {
            "artist": { "type": "text" },
            "song": { "type": "text" },
            "chosen_by": { "type": "keyword" },
            "jukebox_relations": {
                "type": "join",
                "relations": {
                    "artist": "song",
                    "song": "chosen_by"
                }
            }
        }
    }
}

POST jukebox/_create/1
{
    "name": "John Legend",
    "jukebox_relations": { "name": "artist" }
}
POST jukebox/_create/2
{
    "name": "Ariana Grande",
    "jukebox_relations": { "name": "artist" }
}
POST jukebox/_doc/3?routing=1
{
    "song":"All of Me",
    "jukebox_relations": { "name": "song", "parent": 1 }
}
POST jukebox/_doc/4?routing=1
{
    "song": "Beauty and the Beast",
    "jukebox_relations": { "name": "song", "parent": 1 }
}
POST jukebox/_doc/5?routing=2
{
    "song": "Beauty and the Beast",
    "jukebox_relations": { "name": "song", "parent": 2 }
}

POST jukebox/_create/u1?routing=3
{
    "user": "Gabriel",
    "jukebox_relations": { "name": "chosen_by", "parent": 3 }
}
POST jukebox/_create/u2?routing=3
{
    "user": "Berte",
    "jukebox_relations": { "name": "chosen_by", "parent": 3 }
}
POST jukebox/_create/u3?routing=3
{
    "user": "Emma",
    "jukebox_relations": { "name": "chosen_by", "parent": 3 }
}

POST jukebox/_create/u4?routing=4
{
    "user": "Berte",
    "jukebox_relations": { "name": "chosen_by", "parent": 4 }
}
POST jukebox/_create/u5?routing=5
{
    "user": "Emma",
    "jukebox_relations": { "name": "chosen_by", "parent": 5 }
}

Search for all songs (child) of an artist (parent).

GET jukebox/_search
{
    "query": {
        "has_parent": {
            "parent_type": "artist",
            "query": { "match": { "name": "John Legend" } }
        }
    }
}

search for all user-likes (grandchild of the artist) of a song (child of the artist)
GET jukebox/_search
{
    "query": {
        "has_parent": {
            "parent_type": "song",
            "query": {
                "match": { "song": "all of Me" }
            }
        }
    }
}

Searching for all artists (parents) that have one to ten (min_children and max_children) songs
GET jukebox/_search
{
    "query": {
        "has_child": {
            "type": "song",
            "min_children": 1, 
            "max_children": 10, 
            "query": { "match_all": {} }
        }
    }
}

access a child document
GET jukebox/_doc/3?routing=1

* Updating Child Documents
    * One of the significant benefits of a parent-child relationship is the ability to modify a child 
    object independent of the parent.
POST jukebox/_update/5?routing=2
{
    "doc": {
        "song": "Beauty and the Beast (2017)"
    }
}

### Aggregations
* count of user likes and see the user names that liked that song

GET jukebox/_search
{
    "query":{
        "bool":{
            "must": { "match": { "song": "Beauty and the Beast" } },
            "should":{
                "has_child": {
                    "type": "chosen_by",
                    "query": { "match_all":{} },
                    "inner_hits": {}
                }
            }
        }
    },
    "aggs":{
        "user_likes": { "children": { "type": "chosen_by" } }
    }
}