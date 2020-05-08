# object
## single case
1. we want to index document
    * note that in address we will have always single object (not an array)
    ```
    {
        "name": "Michal",
        "surname": "Tumilowicz",
        "address": {
            "street": "Tamka",
            "city": "Warsaw"
        }
    }   
    ```
1. propose mapping as a single object
    ```
    PUT person-object
    {
        "mappings": {
            "properties": {
                "name": { "type": "text" },
                "surname": { "type": "text" },
                ...
            }
        }
    }
    ```
1. index document
    ```
    POST person-object/_create/1
    ```
1. find by each field
    * hint: `query.bool.must.match`, dot notation
    
## array case
1. we want to index document
    * note that in events we will many objects (array)
    ```
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
1. propose mapping with an array
    * mapping is exactly the same as with the single object
    * use term for name
1. find all groups that have events concerning "elasticsearch" and took place in 2018
    * hint: term, range
    
# nested
## single case
1. we want to index document
    * note that in address we will have always single object (not an array)
    ```
    {
        "name": "Michal",
        "surname": "Tumilowicz",
        "address": {
            "street": "Tamka",
            "city": "Warsaw"
        }
    }
   ```
1. propose mapping as a nested object
    ```
    PUT person-nested
    {
        "mappings": {
            "properties": { 
                "name": { "type": "text" },
                "surname": { "type": "text" },
                ...
            }
        }
    }
    ```
1. index document
    ```
    POST person-nested/_create/1
    ```
1. find by each field
    * hint: `query.bool.must.nested` with path
1. find by each field and show nested documents that matches the query
    * hint: `inner_hits`
## array case
1. we want to index document
    * note that in events we will many objects (array)
    ```
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
1. propose mapping with an array
    * mapping is exactly the same as with the single object
    * use term for name
1. index document
    ```
    POST programming-groups-nested/_create/1   
    ```
1. find all groups that have events concerning "elasticsearch" and took place in 2018
    * hint: term, range
1. find all groups that have events with title java and elasticsearch
    * hint: recreate mapping and use `copy_to`
## aggregations
1. run data from `league.md`
1. count players that played at least 30 games for each team 
    * verify outputs when size = 0
    * hint: group by name, nested aggregation, define path, filter with range
1. count teams with at least one player who played at least 30 games
    * verify outputs when size = 0
    * hint: group by name, nested aggregation, define path, filter with range and reverse_nested
    
# join
1. we want to index three types of documents
    * artist
        ```
        {
            "name": "Led Zeppelin"
        }
        ```
    * song
        ```
        {
            "song": "Whole Lotta Love"
            // parent Led Zeppelin
        }
        ```
    * like
        ```
        {
            "user": "Gabriel"
            // liked whole lotta love
        }
        ```
    * relations
        * artist has many songs
        * song has many likes
1. propose mapping as a join
1. fill index with `jukebox.md`
    * note how we are routing appropriate relationships to the same shards
1. update song 'Whole lotta love' to 'Whole Lotta Love' using partial update
    * pay attention to routing
1. verify that name changed
    * hint: get song by id
    * verify, that correct routing is necessary 
1. find all songs of an artist Led Zeppelin
    * hint: `has_parent`, `parent_type`
1. find all users that liked a song
    * hint: `has_parent`, `parent_type`
1. find all artists that have at least one song
    * `has_child`, `type`, `min_children`
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