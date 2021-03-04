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

### Configure JanusGraph to Use the "ConfiguredGraphFactory"

The "ConfiguredGraphFactory" provides methods for dynamically managing the graphs hosted on the server. The default JanusGraph configuration does not enable the ConfiguredGraphFactory. So we need to make some changes to the configuration before we can use it.

The ConfiguredGraphFactory needs to store information about the graphs it is tracking. It uses a graph called the "Configuration Management Graph" to store this information.

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

And there are three changes that need to be made in the YAML configuration file. So let's open it for editing.

```shell
vim conf/gremlin-server/gremlin-server.yaml
```

The three changes are:
1. Add a graph named "ConfigurationManagementGraph" under "graphs".
2. Add a "graphManager" property.
3. Change the value of the "channelizer" property.

```diff
- channelizer: org.apache.tinkerpop.gremlin.server.channel.WebSocketChannelizer
+ channelizer: org.janusgraph.channelizers.JanusGraphWebSocketChannelizer
+ graphManager: org.janusgraph.graphdb.management.JanusGraphManager
  graphs: {
    graph: conf/janusgraph-inmemory.properties,
    graph2: conf/graph2-hbase.properties,
+   ConfigurationManagementGraph: conf/janusgraph-hbase-configurationgraph.properties
  }
```

To apply the new configuration, shutdown the JanusGraph server by pressing CTRL + c if it is already running, then start it again.

```shell
bin/gremlin-server.sh
```

JanusGraph should create the "ConfigurationManagementGraph" on startup. Let's make sure this happened by entering the `list` command in the HBase shell.

```
hbase(main):003:0> list
TABLE
ConfigurationManagementGraph
graph2
2 row(s)
Took 0.0848 seconds
=> ["ConfigurationManagementGraph", "graph2"]
```

### Create a Template and Graphs Dynamically From the Client-Side

Switch to the Gremlin Console. We will need to close the old connection and reconnect because the server was restarted.

```
gremlin> :remote close
==>Removed - Gremlin Server - [localhost/127.0.0.1:8182]
gremlin> :remote connect tinkerpop.server conf/remote.yaml session
==>Configured localhost/127.0.0.1:8182-[be2df6ef-df68-4061-9ee0-c172e40c1d92]
gremlin> :remote console
==>All scripts will now be sent to Gremlin Server - [localhost/127.0.0.1:8182]-[be2df6ef-df68-4061-9ee0-c172e40c1d92] - type ':remote console' to return to local mode
```

Note than this time I connected to the server using a "sessioned connection". This is because it is much easier to build the template configuration map by sending multiple commands to the server.

The following code creates a template configuration. Then creates two graphs: "dynamic1" and "dynamic2" based on this template.

```groovy
templateConfMap = new HashMap();
templateConfMap.put("storage.backend", "hbase");
templateConfMap.put("storage.hostname", "localhost");
ConfiguredGraphFactory.createTemplateConfiguration(new MapConfiguration(templateConfMap));

ConfiguredGraphFactory.create("dynamic1")
ConfiguredGraphFactory.create("dynamic2")
```

Creating these graphs adds the global identifiers "dynamic1" and "dynamic2" for the graph objects. And adds the global identifiers "dynamic1_traversal" and "dynamic2_traversal" for the graph traversal source objects. These identifiers enable clients to access the new graphs.

We cannot use these identifiers right now because we are using a sessioned connection that was established before the identifiers were created. So we need to disconnect from the server and reconnect to bind the new identifiers.

```
:remote close
:remote connect tinkerpop.server conf/remote.yaml
:remote console
```

Now try evaluating the new identifiers to make sure they are defined.

```
gremlin> dynamic1
==>standardjanusgraph[hbase:[localhost]]
gremlin> dynamic1_traversal
==>graphtraversalsource[standardjanusgraph[hbase:[localhost]], standard]
gremlin> dynamic2
==>standardjanusgraph[hbase:[localhost]]
gremlin> dynamic2_traversal
==>graphtraversalsource[standardjanusgraph[hbase:[localhost]], standard]
```

These identifiers will NOT be forgotten when the JanusGraph server is restarted.

Now `list` the tables from the HBase shell to see the tables that JanusGraph created for the new graphs.

```
hbase(main):004:0> list
TABLE
ConfigurationManagementGraph
dynamic1
dynamic2
graph2
4 row(s)
Took 0.0779 seconds
=> ["ConfigurationManagementGraph", "dynamic1", "dynamic2", "graph2"]
```

### Other Functions of the ConfiguredGraphFactory

The ConfiguredGraphFactory can be used to get the list of names of the graphs that it is tracking.

```
gremlin> ConfiguredGraphFactory.getGraphNames()
==>dynamic1
==>dynamic2
```

It can also be used to drop graphs.

```
gremlin> ConfiguredGraphFactory.drop('dynamic2')
==>null
gremlin> ConfiguredGraphFactory.getGraphNames()
==>dynamic1
```
