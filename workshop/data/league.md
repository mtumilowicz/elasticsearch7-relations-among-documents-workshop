1. prepare index
    ```
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
    ```
1. fill with data
    ```
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
    ```
    * note that the team 3 is indexed twice