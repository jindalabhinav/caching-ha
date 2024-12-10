# caching-ha
Uses Redis Sentinel for a Highly Available Caching cluster

## Master-Replicas with Sentinel

When installing the chart with architecture=replication and sentinel.enabled=true, it will deploy a Redis® master StatefulSet (only one master allowed) and a Redis® replicas StatefulSet. In this case, the pods will contain an extra container with Redis® Sentinel. This container will form a cluster of Redis® Sentinel nodes, which will promote a new master in case the actual one fails.

