---
title: Converting JSON query results from Google BigQuery to MySQL
date: 2019-03-24 18:50:37
tags:
---

Have you ever wanted to convert the schema and results passed from the query API of BigQuery into something more acceptable?
There is an endless array of `{ f: { v: } }` nests to contend with sometimes. Well now you can transform it into something
simpler!

<!-- more -->

We are transforming something that looks like this:

``` json
{
    "schema": {
        "fields": [
          {
            "name": "Name",
            "type": "STRING",
            "mode": "NULLABLE"
          },
          {
            "name": "Address",
            "type": "RECORD",
            "mode": "REPEATED",
            "fields": [
              {
                "name": "street",
                "type": "STRING",
                "mode": "NULLABLE"
              },
              {
                "name": "city",
                "type": "STRING",
                "mode": "NULLABLE"
              }
            ]
          }
        ]
      },
      "rows": [
        {
          "f": [
            {
              "v": "Amos"
            },
            {
              "v": [
                {
                  "v": {
                    "f": [
                      {
                        "v": "street1"
                      },
                      {
                        "v": "city1"
                      }
                    ]
                  }
                },
                {
                  "v": {
                    "f": [
                      {
                        "v": "street2"
                      },
                      {
                        "v": "city2"
                      }
                    ]
                  }
                }
              ]
            }
          ]
        }
      ]
}
```

into this?

``` json 
[ { "Name": "Amos",
    "Address": 
     [ { "Address.street": "street1", "Address.city": "city1" },
       { "Address.street": "street2", "Address.city': "city2" } ] } ]
```

Now you can, and for some [very complicated nested results](https://cloud.google.com/bigquery/docs/nested-repeated) as well!

See the *node.js* function below and enjoy! 

``` js
convertBQToMySQLResults(schema, rows) {

    var resultRows = []
    
    function recurse (schemaCur, rowsCur, colName) {

        if (Array.isArray(schemaCur) && !Array.isArray(result[colName])) {
            for(var i=0, l=schemaCur.length; i<l; i++) {
                if (colName === "")
                    recurse(schemaCur[i], rowsCur.f[i], colName + schemaCur[i].name)
                else
                    recurse(schemaCur[i], rowsCur.f[i], colName + "." + schemaCur[i].name)
            }    
        }

        if (schemaCur.type && schemaCur.type === "RECORD") {
            if (schemaCur.mode !== "REPEATED") {
                var valIndex = 0
                for (var p in schemaCur.fields) {
                    if (rowsCur.v === null) {
                        recurse(schemaCur.fields[p], rowsCur, colName + "." + schemaCur.fields[p].name)
                    } else {
                        recurse(schemaCur.fields[p], rowsCur.v.f[valIndex], colName + "." + schemaCur.fields[p].name)
                    }
                    
                    valIndex++
                }
            } 
            
            if (schemaCur.mode === "REPEATED") {   
                result[colName] = [] 
                for (var x in rowsCur.v) {
                    recurse(schemaCur.fields, rowsCur.v[x], colName)
                }
            }
        } else {
            if (schemaCur.mode === "REPEATED") {
                if (rowsCur.v !== null) {
                    result[colName] = rowsCur.v.map( (value, index) => { return value.v })
                } else {
                    result[colName] = [ null ]
                }
                
            } else if (Array.isArray(result[colName])) {
                let nextRow = {} 
                for (var j in schemaCur) {
                    nextRow[colName + "." + schemaCur[j].name] = rowsCur.v.f[j].v 
                }
                result[colName].push(nextRow)
            } else {
                if (colName !== "")
                    result[colName] = rowsCur.v
            }
        }
    }

    for (var r=0, rowsCount=rows.length; r<rowsCount; r++) {
        var result = {};
        recurse(schema, rows[r], "")
        resultRows.push(result)
    }

    return resultRows
}
```
