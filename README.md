* https://www.factweavers.com/blog/join-in-elasticsearch/
* https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-request-body.html#request-body-search-inner-hits
* https://www.waitingforcode.com/elasticsearch/reverse-nested-aggregation-in-elasticsearch/read
* https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html
* https://blog.mimacom.com/parent-child-elasticsearch/

## object
### object
* OBJECTS WORK BEST FOR ONE - TO - ONE RELATIONSHIPS
* You can run queries and aggregations on objects as you would do with flat doc-
  uments. That’s because at the Lucene level they are flat documents.
* No joins are involved. Because everything is in the same document, using
  objects will give you the best performance
* Updating a single object will re-index the whole document.

POST person-object/_create/1
{
    "name": "Michal",
    "surname": "Tumilowicz",
    "address": {
        "street": "Tamka",
        "city": "Warsaw"
    }
}

PUT person-object
{
    "mappings": {
        "properties": { 
            "name": { "type": "text" },
            "surname": { "type": "text" },
            "address": { 
                "properties": {
                    "street": { "type": "text" },
                    "city": { "type": "text" }
                }
            }
        }
    }
}

GET person-object/_search
{
    "query": {
        "bool": {
            "must": [
                { "match": { "name": "Michal" } },
                { "match": { "address.street": "Tamka" } },
                { "match": { "address.city": "Warsaw" } }
            ]
        }
    }
}
### array
POST programming-groups/_create/1
{
    "name": "WJUG",
    "events": [
        {
            "title": "elasticsearch",
            "date": "2019-10-10"
        },
        {
            "title": "java",
            "date": "2018-10-10"
        }
    ]
}

PUT programming-groups
{
    "mappings": {
        "properties": { 
            "name": { "type": "text" },
            "events": { 
                "properties": {
                    "title": { "type": "text" },
                    "date": { "type": "date" }
                }
            }
        }
    }
}

GET programming-groups/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "term": {
                        "events.title": "elasticsearch"
                    }
                },
                {
                    "range": {
                        "events.date": {
                            "from": "2018-01-01",
                            "to": "2019-01-01"
                        }
                    }
                }
            ]
        }
    }
}

## nested
* Internally, nested documents are indexed as different
  Lucene documents
* At the Lucene level, Elasticsearch will index the root document and all the members
  objects in separate documents. But it will put them in a single block
* Documents of a block will always stay together, ensuring they get fetched and queried
  with the minimum number of operations.
* Inner objects must have a nested mapping, to get them indexed as separate
documents in the same block.
* Nested queries and filters must be used to make use of those blocks while
searching.
* There are nested queries,
  filters, and aggregations that help you achieve this. 
  * Running these special queries and
  aggregations will trigger Elasticsearch to join the different Lucene documents within
  the same block and treat the resulting data as the same Elasticsearch document
* Elasticsearch needs to do some extra work to join multiple documents within
  a block
    *  But because of the underlying implementation using blocks, these queries
       and aggregations are much faster than they would be if you had to join completely
       separate Elasticsearch documents
* Because nested documents are
  stuck together, updating or adding one inner document requires re-indexing the
  whole ensemble
* Using the nested-type
  approach, Elasticsearch will have to re-index the group documents with the new event
  and all existing events, which is much slower.
### object
PUT person-nested
{
    "mappings": {
        "properties": { 
            "name": { "type": "text" },
            "surname": { "type": "text" },
            "address": {
                "type": "nested",
                "properties": {
                    "street": { "type": "text" },
                    "city": { "type": "text" }
                }
            }
        }
    }
}

// run this query against person-object
GET person-nested/_search
{
    "query": {
        "bool": {
            "must": [
                { "match": { "name": "Michal" } },
                { 
                    "nested": {
                        "path": "address",
                        "query" : {
                            "bool": {
                                "must": [
                                    { "match": { "address.street": "Tamka" } },
                                    { "match": { "address.city": "Warsaw" } }
                                ]
                            }
                        }
                    }
                }
            ]
        }
    }
}

GET person-nested/_search
{
    "query": {
        "bool": {
            "must": [
                { "match": { "name": "Michal" } },
                { 
                    "nested": {
                        "path": "address",
                        "query" : {
                            "bool": {
                                "must": [
                                    { "match": { "address.street": "Tamka" } },
                                    { "match": { "address.city": "Warsaw" } }
                                ]
                            }
                        },
                        "inner_hits": {}
                    }
                }
            ]
        }
    }
}

### array
PUT programming-groups-nested
{
    "mappings": {
        "properties": { 
            "name": { "type": "text" },
            "eventTitles": { "type": "text" },
            "events": {
                "type": "nested",
                "properties": {
                    "title": { 
                        "type": "text",
                        "copy_to": "eventTitles" 
                    },
                    "date": { "type": "date" }
                }
            }
        }
    }
}

POST programming-groups-nested/_create/1
{
    "name": "WJUG",
    "events": [
        {
            "title": "elasticsearch",
            "date": "2019-10-10"
        },
        {
            "title": "java",
            "date": "2018-10-10"
        }
    ]
}

GET programming-groups-nested/_search
{
    "query": {
        "nested": {
            "path": "events",
            "query": {
                "bool": {
                    "must": [
                        {
                            "term": {
                                "events.title": "elasticsearch"
                            }
                        },
                        {
                            "range": {
                                "events.date": {
                                    "from": "2018-01-01",
                                    "to": "2019-01-01"
                                }
                            }
                        }
                    ]
                }
            }
        }
    }   
}

GET programming-groups-nested/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "term": {
                        "eventTitles": "elasticsearch"
                    }
                },
                {
                    "term": {
                        "eventTitles": "java"
                    }
                }
            ]
        }
    }   d
}

### aggregations
PUT league
{
   "mappings":{
         "properties":{
            "name":{ "type":"keyword" },
            "players":{
               "type":"nested",
               "properties":{
                  "identity":{ "type":"text" },
                  "games":{ "type":"byte" },
                  "nationality":{ "type":"text" }
               }
            }
         }
   }
}

POST league/_doc
{
    "name": "Team 1", 
    "players": [
            {"identity": "Player_1", "games": 30, "nationality": "FR"},
            {"identity": "Player_2", "games": 15, "nationality": "DE"},
            {"identity": "Player_3", "games": 34, "nationality": "FR"},
            {"identity": "Player_4", "games": 11, "nationality": "BR"},
            {"identity": "Player_5", "games": 4, "nationality": "BE"},
            {"identity": "Player_6", "games": 11, "nationality": "FR"}
    ]
}
POST league/_doc
{
    "name": "Team 2", 
    "players": [
        {"identity": "Player_20", "games": 11, "nationality": "FR"},
        {"identity": "Player_21", "games": 15, "nationality": "FR"},
        {"identity": "Player_22", "games": 34, "nationality": "FR"},
        {"identity": "Player_23", "games": 30, "nationality": "FR"},
        {"identity": "Player_24", "games": 4, "nationality": "FR"},
        {"identity": "Player_25", "games": 11, "nationality": "FR"}
    ]
}
POST league/_doc 
{
    "name": "Team 3", 
    "players": [
        {"identity": "Player_30", "games": 11, "nationality": "FR"},
        {"identity": "Player_31", "games": 15, "nationality": "FR"},
        {"identity": "Player_32", "games": 12, "nationality": "FR"},
        {"identity": "Player_33", "games": 15, "nationality": "FR"},
        {"identity": "Player_34", "games": 4, "nationality": "FR"},
        {"identity": "Player_35", "games": 11, "nationality": "FR"}
    ]
}
POST league/_doc
{
    "name": "Team 3", 
    "players": [
        {"identity": "Player_30", "games": 11, "nationality": "FR"},
        {"identity": "Player_31", "games": 15, "nationality": "FR"},
        {"identity": "Player_32", "games": 12, "nationality": "FR"},
        {"identity": "Player_33", "games": 15, "nationality": "FR"},
        {"identity": "Player_34", "games": 4, "nationality": "FR"},
        {"identity": "Player_35", "games": 11, "nationality": "FR"}
    ]
} 

how many players in each team played in at least 30 games?
*     "size": 0, // https://www.elastic.co/guide/en/elasticsearch/reference/current/returning-only-agg-results.html
GET league/_search
{
    "size": 0,
    "aggs":{
        "by_team":{
            "terms":{ "field":"name" },
            "aggs":{
                "at_least_30_games":{
                    "nested":{ "path":"players" },
                    "aggs":{
                        "count_players":{
                            "filter":{
                                "range":{
                                    "players.games":{ "gte":30 }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

how many teams have players who played in at least 30 games?

GET league/_search
{
    "size": 0,
    "aggs":{
        "by_team":{
            "terms":{"field":"name"},
            "aggs":{
                "at_least_30_games":{
                    "nested":{"path":"players"},
                    "aggs":{
                        "count_players":{
                            "filter":{
                                "range":{
                                    "players.games":{"gte":30}
                                }
                            },
                            "aggs":{
                                "team_has_players_at_least_30_games":{
                                    "reverse_nested":{}
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

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

POST /jukebox/_bulk
{"index":{"_id":1}}
{"name":"John Legend","jukebox_relations":{"name":"artist"}}
{"index":{"_id":2}}
{"name":"Ariana Grande","jukebox_relations":{"name":"artist"}}

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

POST jukebox/_bulk?routing=3
{"index":{"_id":"l-1"}}
{"user":"Gabriel","jukebox_relations":{"name":"chosen_by","parent":3}}
{"index":{"_id":"l-2"}}
{"user":"Berte","jukebox_relations":{"name":"chosen_by","parent":3}}
{"index":{"_id":"l-3"}}
{"user":"Emma","jukebox_relations":{"name":"chosen_by","parent":3}}

POST jukebox/_create/l-4?routing=4
{
    "user": "Berte",
    "jukebox_relations": { "name": "chosen_by", "parent": 4 }
}
POST jukebox/_create/l-5?routing=5
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