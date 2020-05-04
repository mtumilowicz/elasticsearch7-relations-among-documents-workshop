* https://www.factweavers.com/blog/join-in-elasticsearch/
* https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-request-body.html#request-body-search-inner-hits
* https://www.waitingforcode.com/elasticsearch/reverse-nested-aggregation-in-elasticsearch/read

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
    }   
}