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
    "name": "Led Zeppelin",
    "jukebox_relations": { "name": "artist" }
}

POST jukebox/_create/2
{
    "name": "Sandy Denny",
    "jukebox_relations": { "name": "artist" }
}

POST jukebox/_doc/3?routing=1
{
    "song": "Whole lotta love",
    "jukebox_relations": { "name": "song", "parent": 1 }
}

POST jukebox/_doc/4?routing=1
{
    "song": "Battle of Evermore",
    "jukebox_relations": { "name": "song", "parent": 1 }
}

POST jukebox/_doc/5?routing=2
{
    "song": "Battle of Evermore",
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