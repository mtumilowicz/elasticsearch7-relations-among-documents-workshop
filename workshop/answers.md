# object
## single case
1. propose mapping as a single object
    ```
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
    ```
1. index document
    ```
    POST person-object/_create/1
    {
        "name": "Michal",
        "surname": "Tumilowicz",
        "address": {
            "street": "Tamka",
            "city": "Warsaw"
        }
    }
    ```
1. find by each field
    ```
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
    ```
## array case
1. propose mapping with an array
    ```
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
    ```
1. find all groups that have events concerning "elasticsearch" and took place in 2018
    ```
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
    ```

# nested
## single case
1. propose mapping as a nested object
    ```
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
    ```
1. index document
    ```
    POST person-nested/_create/1
    {
        "name": "Michal",
        "surname": "Tumilowicz",
        "address": {
            "street": "Tamka",
            "city": "Warsaw"
        }
    }
    ```
1. find by each field
    ```
    GET person-nested/_search
    {
        "query": {
            "bool": {
                "must": [
                    { "match": { "name": "Michal" } },
                    { "nested": {
                        "path": "address",
                        "query" : {
                            "bool": {
                                "must": [
                                    { "match": { "address.street": "Tamka" } },
                                    { "match": { "address.city": "Warsaw" } }
                                ]
                            }
                        }
                    } }
                ]
            }
        }
    }
    ```
1. find by each field and show nested documents that matches the query
    ```
    GET person-nested/_search
    {
        "query": {
            "bool": {
                "must": [
                    { "match": { "name": "Michal" } },
                    { "nested": {
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
                    } }
                ]
            }
        }
    }
    ```
## array case
1. propose mapping with an array
    ```
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
    ```
1. index document
    ```
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
    ```
1. find all groups that have events concerning "elasticsearch" and took place in 2018
    ```
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
    ```
1. find all groups that have events with title java and elasticsearch
    ```
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
        }
    }   
    ```
## aggregations
1. run data from `league.md`
1. count players that played at least 30 games for each team 
    ```
    GET league/_search
    {
        "size": 0,
        "aggs": {
            "by_team": {
                "terms": { "field":"name" },
                "aggs": {
                    "at_least_30_games": {
                        "nested": { "path": "players" },
                        "aggs": {
                            "count_players": {
                                "filter":{ "range": { "players.games": { "gte": 30 } } }
                            }
                        }
                    }
                }
            }
        }
    }
    ```
1. count teams with at least one player who played at least 30 games
    ```
    GET league/_search
    {
        "size": 0,
        "aggs": {
            "by_team": {
                "terms": { "field": "name" },
                "aggs": {
                    "at_least_30_games": {
                        "nested": { "path": "players" },
                        "aggs": {
                            "count_players": {
                                "filter": { "range":{ "players.games": {"gte": 30} } },
                                "aggs": {
                                    "team_has_players_at_least_30_games": {
                                        "reverse_nested": {}
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    ```

# join
1. propose mapping as a join
    ```
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
    ```
1. update song 'Whole lotta love' to 'Whole Lotta Love' using partial update
    ```
    POST jukebox/_update/3?routing=1
    {
        "doc": {
            "song": "Whole Lotta Love"
        }
    }
    ```
1. verify that name changed
    ```
    GET jukebox/_doc/3?routing=1
    ```
1. find all songs of an artist Led Zeppelin
    ```
    GET jukebox/_search
    {
        "query": {
            "has_parent": {
                "parent_type": "artist",
                "query": { "match": { "name": "Led Zeppelin" } }
            }
        }
    }
    ```
1. find all users that liked a song
    ```
    GET jukebox/_search
    {
        "query": {
            "has_parent": {
                "parent_type": "song",
                "query": {
                    "match": { "song": "Whole Lotta Love" }
                }
            }
        }
    }
    ```
1. find all artists that have at least one song
    ```
    GET jukebox/_search
    {
        "query": {
            "has_child": {
                "type": "song",
                "min_children": 1, 
                "query": { "match_all": {} }
            }
        }
    }
    ```
## aggregations
1. count user likes for given song and show users that liked that song
    ```
    GET jukebox/_search
    {
        "query": {
            "bool": {
                "must": { "match": { "song": "Battle of Evermore" } },
                "should": {
                    "has_child": {
                        "type": "chosen_by",
                        "query": { "match_all": {} },
                        "inner_hits": {}
                    }
                }
            }
        },
        "aggs":{
            "user_likes": { "children": { "type": "chosen_by" } }
        }
    }
    ```