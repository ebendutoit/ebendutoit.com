---
title: Convert BigQuery response to MYSQL in python 3
date: 2020-10-02 12:55:56
tags:
---

More than a year ago I wrote a post with a Javascript (node JS) function inside it that shows how we can recursively parse BigQuery rows results from their API into a flattened "mysql"-like JSON file. Recently I needed the python version of this function, so I provide it here. Enjoy!

``` python
def convert_bq_rows_to_json(self, schema, rows):

        result_rows = []
        result = {}
        def recurse (schema_cur, rows_cur, col_name):

            if isinstance(schema_cur, list):
                for i in range(0, len(schema_cur)):
                    print(schema_cur[i])
                    if (col_name == ""):
                        recurse(schema_cur[i], rows_cur['f'][i], col_name + schema_cur[i]['name'])
                    else:
                        recurse(schema_cur[i], rows_cur['f'][i], col_name + "." + schema_cur[i]['name'])
                
            if 'type' in schema_cur and schema_cur['type'] == "RECORD":
                if 'mode' in schema_cur and schema_cur['mode'] != "REPEATED":
                    val_index = 0
                    for p in schema_cur['fields']:
                        if rows_cur['v'] == None:
                            recurse(schema_cur['fields'][p], rows_cur, col_name + "." + schema_cur['fields'][p]['name'])
                        else:
                            recurse(schema_cur['fields'][p], rowsCur.v.f[valIndex], colName + "." + schemaCur.fields[p].name)
                    
                        val_index+=1
                    
                if (schema_cur['mode'] == "REPEATED"):  
                    result[colName] = [] 
                    for x in rows_cur['v']: 
                        recurse(schema_cur['fields'], rows_cur['v'][x], col_name)
            else:
                if 'mode' in schema_cur and schema_cur['mode'] == "REPEATED":
                    if rows_cur['v'] != None:
                        result[col_name] = map(lambda x : x['v'], rows_cur['v'])
                    else:
                        result[col_name] = [ None ]
                    
                elif col_name in result and type(result[col_name]) is list:
                    next_row = {} 
                    for j in schema_cur:
                        next_row[col_name + "." + schema_cur[j].name] = map(lambda x: x['v'], rows_cur['v']['f'][j]['v']) if type(rows_cur['v']['f'][j]['v']) is list else rows_cur['v']['f'][j]['v']
                    
                    result[col_name].append(next_row)
                else:
                    if col_name != "":
                        result[col_name] = rows_cur['v']

        for r in range(0, len(rows)):
            recurse(schema['fields'], rows[r], "")
            result_rows.append(result)
        
        return result_rows

```
