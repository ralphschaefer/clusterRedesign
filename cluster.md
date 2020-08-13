# akka cluster

--
* Technologie suitable for redesigning a monolithic application ?
* Short explanation of akka cluster "artifacts" and patterns
* Possible usecases for the "Simplifer"

 

---

## Cluster patterns

--
* [CQRS](https://m.heise.de/developer/artikel/CQRS-neues-Architekturprinzip-zur-Trennung-von-Befehlen-und-Abfragen-1797489.html?seite=all) !!!

--
* Event Sourcing

---

## Cluster "artifacts" 

--
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

---
### serialization
Messages in between the nodes must be serialized. Native JAVA serialzation is known to be buggy.

Alternatives [are](https://manuel.bernhardt.io/2018/07/20/akka-anti-patterns-java-serialization/):
* _avro_ (has json-schemas abilities, for example [akka-avro-serializer](https://github.com/hopped/akka-avro-serializer) )
  Avro can keep the schemas in a "Schema Registry", which is capable of handling schema versions.
* _kryo_ ( lightwight, [twitter-chill](https://github.com/twitter/chill))

---
### monitoring
- What to monitor -> _metric_
- How to collect the metric -> _timeline DB_ (example [prometheus](https://prometheus.io/))
- displaying the metrics timeline -> _visualisation_ (example [grafana](https://grafana.com/))
- notification if a metric runs out of defined boundaries -> _alerting_ (example [grafana alerting](https://grafana.com/docs/grafana/latest/alerting/notifications/))

--
#### how to get "cheap metrics"
- not for free, but very useful: [Kamon](https://kamon.io/). This provides cluster metrics
and JVM metrics from each node. Works with java aspectJ (they maybe have changed this meanwhile.. ?)
the kamon metrics can later also be used by _prometeus_ for scraping
- write your own :-)

--
#### how to get "cheap cheap timeline DB and visualistion"
- also: use Kamon SAAS backend 

---
### message queue

--
**short notes about akka cluster management**:

--
* The cluster is kept intact by sending [heartbeats](https://doc.akka.io/docs/akka/current/cluster-usage.html#failure-detector)
in between the nodes. If these _heartbeats_ are disturbed, the node will leave the cluster, or 
the cluster breaks apart in many sub-clusters ([split-brain](https://en.wikipedia.org/wiki/Split-brain_(computing) )   
A very common way to disturb these _hearbeats_ is to send very large messages, or to send messages more frequent as physically possible.

--

**THEREFORE**:

* An other mechanism should be used for big and frequent messages: eg. [Kafka](https://kafka.apache.org/). An easy 
way to get this up & running it the [Confluent platform](https://www.confluent.io/); once
again: not for free, but very useful. 

---
**There are certain other advantages of kafka:**
* custom connector 
* serialization
* aggregation
* ready made metrics for monitoring
* Almost any cloud provider has got kafka
* messages are not event based, but polling is used, so the load on each node it limited
* timeline database layout, that is perfect for logging
* and much more...

 
---
### persistence
- master / master replication should be possible
- or at least if only master / slave is possible, _command_ and _query_ access to 
db should be separated 

---
### distribute data
to distribute data, [CRDTs](https://m.heise.de/developer/artikel/Verteilte-Daten-ohne-Muehe-Conflict-Free-Replicated-Data-Types-3944421.html?seite=all)
are a possible pattern. A _CRDT_ is the atomic part of distributed data. _CRDT_ are mergeable 
without producing conflicts. 

*Distributed Data in Akka...*  
* is eventually consistent
* will be distributed via gossip protocol
* are not suited for "Big Data"
* can be self implemented
* has several predefined DataType (Maps, Sets, Counters, etc...)

---
## cluster "scaling"

--
* Manual scaling is done by adding or removing a node to the cluster.

--
* Automatic scaling needs [metric(s)](#monitoring). Based on these metrics the cluster 
can be scaled up & down automatically. Every cloud provider has got its own system 
of automatic scaling, but meanwhile almost all of them are providing 
[kubernets system](https://kubernetes.io/de/). 

---
## Further Reading
* [Reactive web application](https://www.amazon.de/Reactive-Web-Applications-Covers-Streams/dp/163343009X/ref=sr_1_2?__mk_de_DE=%C3%85M%C3%85%C5%BD%C3%95%C3%91&dchild=1&keywords=Reactive+Web+Applications&qid=1596694303&sr=8-2)

---
# the "simplifier" usecase (IMHO) for cluster redesign (by step) 

---
### Step 1
* create different domains
    * connector
    * database access
    * logging
    * authorisation scheme
    * plugins
    * ui
    * api
    * ...

--
* create nodes accordingly

--
* create ``docker-compose`` setup for local deployment for dev

---

### Step 2
* implement monitoring subsystem
* 1st with kamon
* then replace this with prometheus and grafana

--

### Step 3
* redesign logging (based on kafka)
    * create logging consumer and producers
    * create logging consumer for customers with filters
    * create kafka logging connector to enable customer logging 
--

### Step 3
* redesign connectors (async, result based on kafka)
    * all event based connectors 
    * all database base connector
    * create a "kafka connect" connector
---

### Step 4
* change db access method (depends on if master / master replication is possible)
    * all other decisions depends on which db to use
--

### Step 5
* build kubernetes deployment
    * ...
--

### Step 6
* build automatic scaling    
    * ...
