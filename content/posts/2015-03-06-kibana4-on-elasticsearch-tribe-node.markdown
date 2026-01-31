---
layout: post
comments: true
title:  "How to install and run Kibana4 on a elasticsearch tribe node"
date:   2015-03-06 19:19:43
categories: kibana elasticsearch
---

Not long ago Kibana 4 was released and I think it's a major improvement in relation to the previos version of Kibana. Some changes worth mentioning is that Kibana 4 now runs a node js web server, Express, while Kibana3 was just plain js files that you had to serve in a httpd or nginx. This really is a great improvement! In kibana3 elasticsearch web port 9200 had to be reachable by the client, not an ideal solution as this is something you want to limit as it enables anyone to just do a curl -XDELETE. The workaround was to make a lot of proxy rewrites in the webserver serving kibana - Workable but felt more of a hack than a good solution. Running kibana server side reduces that hassle. <!-- more -->

Other changes is a completely remade UI, it definitely has more of a Splunk feeling to it. But best is probably the great visualization features Kibana4 introduces powered by the newly introduced aggregations in Elasticsearch 1.4. The only downside with Kibana4 I could find is that you cannot run it on a Elasticsearch tribe node, it needs write access to elasticsearch while a tribe node is read-only. Shame really, a tribe node is very useful for example when you have to separeate ES clusters (one each in different data centers would be the most usual use-case I guess), and you want to federate/unite the results from both the clusters with Kibana. From the ES docs:

> The tribes feature allows a tribe node to act as a federated client across multiple clusters. The tribe node works by retrieving the cluster state from all connected clusters and merging them into a global cluster state.

Just add this in elasticsearch.yml on the machine that the tribe node is running on:

```
tribe:
    t1: cluster.name:   cluster_one
    t2: cluster.name:   cluster_two
```


Long story short, with some patience and Chrome I found a way to make Kibana4 work running against a ES tribe node. Basicly Kibana4 will throw an error every time it tries to write to ES. I intercepted each of these calls and executed them manually on the master in one of the clusters. It has worked great for me so far but I cannot guarantee anything. Here goes:

```javascript
curl -X PUT http://localhost:9200/.kibana/
curl -H 'Accept: application/json' -X PUT -d '{"index-pattern":{"properties":{"title":{"type":"string"},"timeFieldName":{"type":"string"},"intervalName":{"type":"string"},"customFormats":{"type":"string"},"fields":{"type":"string"}}}}' localhost:9200/.kibana/_mapping/index-pattern
curl -H 'Accept: application/json' -X PUT -d '{"search":{"properties":{"title":{"type":"string"},"description":{"type":"string"},"hits":{"type":"integer"},"columns":{"type":"string"},"sort":{"type":"string"},"version":{"type":"integer"},"kibanaSavedObjectMeta":{"properties":{"searchSourceJSON":{"type":"string"}}}}}}' localhost:9200/.kibana/_mapping/search
curl -H 'Accept: application/json' -X PUT -d '{"dashboard":{"properties":{"title":{"type":"string"},"hits":{"type":"integer"},"description":{"type":"string"},"panelsJSON":{"type":"string"},"version":{"type":"integer"},"kibanaSavedObjectMeta":{"properties":{"searchSourceJSON":{"type":"string"}}}}}}' localhost:9200/.kibana/_mapping/dashboard
curl -H 'Accept: application/json' -X PUT -d '{"visualization":{"properties":{"title":{"type":"string"},"visState":{"type":"string"},"description":{"type":"string"},"savedSearchId":{"type":"string"},"version":{"type":"integer"},"kibanaSavedObjectMeta":{"properties":{"searchSourceJSON":{"type":"string"}}}}}}' localhost:9200/.kibana/_mapping/visualization

```
