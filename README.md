* https://www.factweavers.com/blog/join-in-elasticsearch/


## object
### object
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