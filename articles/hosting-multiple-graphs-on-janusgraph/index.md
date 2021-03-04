# Hosting Multiple Graphs on JanusGraph

### Introduction

In this article, I will explain how to configure JanusGraph to host multiple graphs on the same JanusGraph instance. I will start by explaining how you can add graphs from the JanusGraph configuration files. Then I will explain how to configure JanusGraph to use the `ConfiguredGraphFactory` so you can add and remove graphs dynamically without having to edit the configuration files or restart the JanusGraph server.

### View the Default Graph Configuration

Out of the box, JanusGraph is configured to host only one graph. And to use the in-memory storage backend to save this graph data. Looking at this default graph configuration will help us determine what we need to do to add more graphs.

Go to the JanusGraph root folder and open the file "conf/gremlin-server/gremlin-server.yaml".

```shell
cd /opt/janusgraph-0.5.3/
vim conf/gremlin-server/gremlin-server.yaml
```

You will see that this configuration file has a "graphs" list containing one "graph" item.

```yaml
graphs: {
  graph: conf/janusgraph-inmemory.properties
}
```

The key "graph" is the identifier that can be used by remote clients to access this graph. And the value is a path to a properties/configuration file that specifies things like the storage backend used to save this graph data and the mixed index backend.

Next, open the file "scripts/empty-sample.groovy".

```shell
vim scripts/empty-sample.groovy
```

The last line in this file adds the identifier "g" and its value (the graph traversal source) to the globals map.

```groovy
globals << [g : graph.traversal()]
```

So we should be able to use the identifiers "graph" and "g" from a remote client to access the graph and the graph traversal source respectively. Let's try this!

Start the JanusGraph server.

```shell
bin/gremlin-server.sh
```

And from a different terminal window start the Gremlin Console.

```shell
bin/gremlin.sh
```

Then connect the Gremlin Console to the Gremlin server.

```
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured localhost/127.0.0.1:8182
gremlin> :remote console
==>All scripts will now be sent to Gremlin Server - [localhost/127.0.0.1:8182] - type ':remote console' to return to local mode
```

And try evaluating the identifiers "graph" and "g".

```
gremlin> graph
==>standardjanusgraph[inmemory:[127.0.0.1]]
gremlin> g
==>graphtraversalsource[standardjanusgraph[inmemory:[127.0.0.1]], standard]
```

They point to the graph and the graph traversal source just as expected.

Let's add a second graph and instruct JanusGraph to save the graph data in HBase.

### Start HBase

Navigate to the HBase root folder and start the HBase server.

```shell
cd opt/hbase-2.1.5/
bin/start-hbase.sh
```

And start the HBase shell to see the tables that JanusGraph creates in HBase.

```shell
bin/hbase shell
```

You can run the `list` command to see the tables that you have initially. I do not have any tables as shown in the shell output below.

```
hbase(main):001:0> list
TABLE
0 row(s)
Took 0.3452 seconds
=> []
```

### Add a Graph From the Configuration

We need to create a properties file that specifies the properties of the second graph we want to create.

Navigate to the JanusGraph root folder, copy the file "conf/janusgraph-hbase.properties", then open the copy for editing.

```shell
cd /opt/janusgraph-0.5.3/
cp conf/janusgraph-hbase.properties conf/graph2-hbase.properties
vim conf/graph2-hbase.properties
```

I will add only one line to this properties file to specify the name of the HBase table that will be used to store the graph data.

```properties
storage.hbase.table = graph2
```

Now open the "conf/gremlin-server/gremlin-server.yaml" file for editing.

```shell
vim conf/gremlin-server/gremlin-server.yaml
```

And add a second item under "graphs". The key/identifier will be "graph2" and the value will be the path to the properties file we just created.

```diff
graphs: {
  graph: conf/janusgraph-inmemory.properties,
+ graph2: conf/graph2-hbase.properties
}
```

Then open the file "scripts/empty-sample.groovy" for editing.

```shell
vim scripts/empty-sample.groovy
```

And add a new graph traversal source to the `globals` map.

```diff
globals << [g : graph.traversal()]
+ globals << [g2 : graph2.traversal()]
```

To apply the new configuration, shutdown the JanusGraph server by pressing CTRL + c if it is already running, then start the server again.

```shell
bin/gremlin-server.sh
```

JanusGraph should create the HBase table for the new graph on startup. Use the HBase shell to see if this table was created successfully.

```
hbase(main):002:0> list
TABLE
graph2
1 row(s)
Took 0.2073 seconds
=> ["graph2"]
```

Now go to the Gremlin Console. Since we restarted the server, we will need to close the connection from the console then reconnect.

```
gremlin> :remote close
==>Removed - Gremlin Server - [localhost/127.0.0.1:8182]
gremlin> :remote connect tinkerpop.server conf/remote.yaml
==>Configured localhost/127.0.0.1:8182
gremlin> :remote console
==>All scripts will now be sent to Gremlin Server - [localhost/127.0.0.1:8182] - type ':remote console' to return to local mode
```

And let's test the identifiers "graph2" and "g2" to see if they are recognized.

```
gremlin> graph2
==>standardjanusgraph[hbase:[127.0.0.1]]
gremlin> g2
==>graphtraversalsource[standardjanusgraph[hbase:[127.0.0.1]], standard]
```

They are indeed recognized. So we successfully added a graph from the configuration. Which is great! But note that we had to edit the configuration files and restart the server. Depending on your use case, this may or may not be acceptable. In the next sections, I will show how to add and remove graphs dynamically without service interruptions.

### Configure JanusGraph to User the "ConfiguredGraphFactory"

The "ConfiguredGraphFactory" provides methods for dynamically managing the graphs hosted on the server. The default JanusGraph configuration does not enable the ConfiguredGraphFactory. So we need to make some changes to the configuration before we can use it.

The ConfiguredGraphFactory needs to store information about the graphs it is tracking. It uses a graph (sometimes referred to as the "Configuration Management Graph") to store this information.

We need a properties file for this configuration management graph. JanusGraph comes with two properties files that can be used for this purpose: "janusgraph-cassandra-configurationgraph.properties" and "janusgraph-cql-configurationgraph.properties". But I want to store the configuration graph in HBase. So I will create a new file "janusgraph-hbase-configurationgraph.properties" by copying one of the two files that came with JanusGraph. Then I will open the new file for editing.

```shell
cp conf/janusgraph-cql-configurationgraph.properties conf/janusgraph-hbase-configurationgraph.properties
vim conf/janusgraph-hbase-configurationgraph.properties
```

There is only one line that I need to change in the new properties file to set the storage backend to "hbase" instead of "cql".

```diff
- storage.backend=cql
+ storage.backend=hbase
```




