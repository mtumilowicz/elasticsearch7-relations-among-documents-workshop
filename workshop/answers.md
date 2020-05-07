# object

## single case
1. we want to index document like that
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
1. map as a single object
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
1. we want to index document like that
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
1. try to map it exactly like with the single case is not proper
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