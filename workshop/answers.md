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