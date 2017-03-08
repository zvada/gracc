# Direct Access

GRACC runs on [Elasticsearch](https://www.elastic.co/products/elasticsearch),
which can be accessed via many [client libraries](https://www.elastic.co/guide/en/elasticsearch/client/community/current/index.html) and tools. 

A read-only endpoint into the GRACC Elasticsearch is available at https://gracc.opensciencegrid.org/q.

## cURL

The Elasticsearch [query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/query-dsl.html) is quite complex, but is also very powerful.
Below is a relatively simple but non-trivial example of directly querying GRACC from the 
command-line via cURL. This query will calculate the number of payload jobs run by 
LIGO in January 2017, and the total wall time they used. 

```
curl 'https://gracc.opensciencegrid.org/q/gracc.osg.summary/_search?pretty' --data-binary '
{
    "query": {
        "query_string": {
            "query": "VOName:ligo AND ResourceType:Payload AND EndTime:[2017-01-01 TO 2017-02-01]"
        }
    },
    "aggs": {
        "walltime": {
            "sum": {
                "field": "CoreHours"
            }
        },
        "jobs": {
            "sum": {
                "field": "Njobs"
            }
        }
    },
    "size": 0
}'
```

Which might return:
```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 6,
    "successful" : 6,
    "failed" : 0
  },
  "hits" : {
    "total" : 68,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "jobs" : {
      "value" : 20626.0
    },
    "walltime" : {
      "value" : 41323.43916666666
    }
  }
}
```

Compare to the [VO Summary](https://gracc.opensciencegrid.org/dashboard/db/vo-summary?from=1483228800000&to=1485907200000&var-interval=$__auto_interval&var-vo=ligo&var-type=Payload) dashboard in Grafana.
