# akka cluster: self organizing framework to build cluster

## Cluster patterns
* [CQRS](https://m.heise.de/developer/artikel/CQRS-neues-Architekturprinzip-zur-Trennung-von-Befehlen-und-Abfragen-1797489.html?seite=all) !!!
* Event Sourcing

## Cluster "artifacts" 

### serialization
Messages in between the nodes must be serialized. Native JAVA serialzation is known to be buggy.

Alternatives [are](https://manuel.bernhardt.io/2018/07/20/akka-anti-patterns-java-serialization/):
* _avro_ (has json-schemas abilities, for example [akka-avro-serializer](https://github.com/hopped/akka-avro-serializer) )
  Avro can keep the schemas in a "Schema Registry", which is capable of handling schema versions.
* _kryo_ ( lightwight, [twitter-chill](https://github.com/twitter/chill))

   
### nodes

Nodes are computation units in which actor system(s) are available for the cluster. 
The actor system in a node should represent only on single domain (example: database, plugin, user computation, etc.)
* node discovery is done by the akka subsystem
* node administration must not be done by akka subsystem
* nodes with the same domain can be run in parallel. messages _can_ be distributed by the akka 
system in this domain (via LB, round robbin, any other metric) 
  
### the "seednode" ...
is obsolet (IMHO), there are much better akka discovery feature to find all cluster 
nodes (kubernetes, aws, dns, or custom made)

### monitoring
- What to monitor -> _metric_
- How to collect the metric -> _timeline DB_ (example [prometheus](https://prometheus.io/))
- displaying the metrics timeline -> _visualisation_ (example [grafana](https://grafana.com/))
- notification if a metric run out of defined boundaries -> _alerting_ (example [grafana alerting](https://grafana.com/docs/grafana/latest/alerting/notifications/))

#### how to get "cheap metrics"
- not for free, but very usefull: [Kamon](https://kamon.io/). This provides cluster metrics
and JVM metrics from each node. Works with java aspectJ (maybe the have changed it meanwhile.. ?)
the kamon metrics can later also be used by _prometeus_ for scraping
- write your own :-)

#### how to get "cheap cheap timeline DB and visualistion"
- also use Kamon SAAS backend 


### message queue
**short notes about akka cluster management**:
 
The cluster is kept intact by sending [heartbeats](https://doc.akka.io/docs/akka/current/cluster-usage.html#failure-detector)
in between the nodes. If these _heartbeats_ are disturbed, the node leave the cluster, or 
the cluster breaks apart in many sub-clusters ([split-brain](https://en.wikipedia.org/wiki/Split-brain_(computing) ))   
A very common way to disturb these _hearbeats_ is to send very large messages, or to send messages more frequent as it 
is physically possible.

**Therfore**:

An other mechanism should be uses: eg. [Kafka](https://kafka.apache.org/). An easy 
way to get this up & running it the [Confluent platform](https://www.confluent.io/) once
again: not for free, but very useful. 

There are certain other advantages of kafka:
* custom connector 
* serialsation
* aggregation
* ready made metrics for monitoring
* Almost any cloud provider has got kaffka
* mesages are not event based, but polling is used, so the load on each node it limited
* timeline database layout, that is perfect for logging
* and much more...
 
### persistence
- master / master replication should be possible
- or at least if only master / slave is possible, _command_ and _query_ access to 
db should be seperated 

## cluster "scaling"
Manual scaling is done by adding or removing a node to the cluster.

Atomatic scaling needs [metric(s)](#monitoring). Based on these metrics the cluster 
can be scaled up & down automatically. Every cloud provider has got its own system 
of automatic scaling, but meanwhile almost all of them are providing 
[kubernets system](https://kubernetes.io/de/). 

## Further Reading
* [Reactive web application](http://file.allitebooks.com/20161008/Reactive%20Web%20Applications.pdf)


# the "simplifier" usecase (IMHO) for cluster redesign (by step) 

* create different domains
    * connector
    * database access
    * logging
    * authorisation scheme
    * plugins
    * ui
    * api
* create nodes accordingly
* create ``docker-compose`` local deployment for dev
------------------------------------------------
* implement monitoring subsystem
------------------------------------------------
* redesign logging (based on kafka)
------------------------------------------------
* redesign connectors (async, result based on kafka)
------------------------------------------------
* change db access method (depends on if master / master replication is possible)
-----------------------------------------------
* build kubernetes deployment
-----------------------------------------------
* build automatic scaling    

