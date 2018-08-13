Sometimes you'll have your ElasticSearch cluster health not ok, since some log shards can get lost and never arrive to any node. Here's how to quickly reroute them and restore the cluster health.

First, check the cluster status.

```
~ # curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'

{
  "cluster_name" : "logsearch-cluster",
  "status" : "red",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 646,
  "active_shards" : 1292,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 4,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 99.69135802469135
}
```

Then, check the log shards left unassigned.

`~ # curl -s 'http://localhost:9200/_cat/shards' | grep UNASSIGNED`

Note the numerical value next to each file entry. In this case, it was 3 and the unassigned sharded log file in the shard was haproxy-2016.08.27.
Now we will re-route the unassigned shards to any of the three available LogSearch clustered hosts. We will issue this JSON POST request and complete it with the data we've gathered so far.

```
~ # curl -XPOST -d '{ "commands" : [ {
  "allocate" : {
       "index" : "haproxy-2016.08.27",
       "shard" : 3,
       "node" : "logsearch2",
       "allow_primary":true
     }
  } ] }' http://localhost:9200/_cluster/reroute?pretty
```

Change index key with the log filename listed as UNASSIGNED. Note that you can use logsearch1/2/3 as a value, these being example hostnames for a three-node cluster. Alternatively, try shard 0 if it's not being processed.

After the task is finished, check if there are any missing steps with

`~ # curl -s 'http://localhost:9200/_cat/shards' | grep UNASSIGNED`

again and repeat step 3 until the command's output is blank.

Check that the cluster status is in GREEN.

```
~ # curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'

{
  "cluster_name" : "logsearch-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 658,
  "active_shards" : 1316,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_m"
}
```

In case you have multiple shards to reroute, here's a Python script for going through all of them (uses elasticsearch-py >=2.0).
