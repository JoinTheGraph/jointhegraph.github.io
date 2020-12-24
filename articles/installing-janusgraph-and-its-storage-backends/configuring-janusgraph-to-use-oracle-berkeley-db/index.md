## Installing JanusGraph and Its Storage Backends

# 2. Configuring JanusGraph to Use Oracle Berkeley DB

### Introduction

This is the second article in a series of articles about installing and configuring JanusGraph and its storage backends on a Linux Ubuntu server.

In this article, I will explain how to configure JanusGraph to use "Oracle Berkeley DB Java Edition" as its storage backend.

### About Oracle Berkeley DB

Oracle Berkeley DB is an embedded key-value database library. The words "embedded" and "library" mean that Berkeley DB will be loaded and executed in the JanusGraph process as opposed to running in a separate process. So Berkeley DB is a database management library, not a stand-alone database server like Cassandra or HBase.

The Berkeley DB JAR file is included in the JanusGraph package that we downloaded in the previous article. So there is no need to download or install Berkeley DB separately.

### Configure JanusGraph to Use Berkeley DB

Open the file "conf/gremlin-server/gremlin-server.yaml" for editing (I suggest you backup the original file before making any changes). Then change the "graph" configuration value from

```yaml
graphs: {
  graph: conf/janusgraph-inmemory.properties
}
```

to

```yaml
graphs: {
  graph: conf/janusgraph-berkeleyje-lucene.properties
}
```

Now the yaml file is pointing to another configuration file "conf/janusgraph-berkeleyje-lucene.properties" which tells JanusGraph to use "Oracle Berkeley DB Java Edition" as the storage backend. And "Apache Lucene" as the mixed index backend. In the next section, I will make a couple of small tweaks to this "properties" file.

### Change the Data and Index Storage Directories

Open the file "conf/janusgraph-berkeleyje-lucene.properties". There is a line in this file that specifies the directory where Berkeley DB should place the database data files.

```properties
storage.directory=../db/berkeley
```

All paths are relative to the JanusGraph root folder. So the default configuration will place the database data files outside the JanusGraph root folder. This will not work for me because I like to run JanusGraph with a special user that is not allowed to write outside the JanusGraph root folder. So I will replace the `..` in the path by `.` to place the database folder and files under the JanusGraph folder.

```properties
storage.directory=./db/berkeley
```

I also need to change the path to the mixed index data directory. So I will change the configuration line

```properties
index.search.directory=../db/searchindex
```

to

```properties
index.search.directory=./db/searchindex
```

Now the JanusGraph server process will not need to write anything outside the JanusGraph root folder.

### Run and Test the JanusGraph Server

Execute the shell script "bin/gremlin-server.sh" to start the JanusGraph server. Then open another terminal window and execute the shell script "bin/gremlin.sh" to start the Gremlin console.

Type this command in the Gremlin console to connect it to the server.

```
:remote connect tinkerpop.server conf/remote.yaml
```

Then type this command to send all the following commands to the server without having to precede them with `:>`

```
:remote console
```

The server should create a `graph` and a `g` object based on the configuration. Let's make sure these variables are defined.

```
gremlin> graph
==>standardjanusgraph[berkeleyje:./db/berkeley]
gremlin> g
==>graphtraversalsource[standardjanusgraph[berkeleyje:./db/berkeley], standard]
```

The string representation of the graph object indicates that it is using Berkeley DB and the storage directory that we specified in the configuration. Perfect!

Let's make sure that the graph is initially empty.

```
gremlin> g.V().count()
==>0
```

Then add a couple of vertices

```
gremlin> g.addV('person').property('name', 'p1')
==>v[4136]
gremlin> g.addV('person').property('name', 'p2')
==>v[4152]
```

### Ensure That the Graph Data Is Persisted

Switch to the terminal that was used to start the JanusGraph server and press CTRL + c to shutdown the server. Then run "bin/gremlin-server.sh" to start it again.

Go to the Gremlin Console terminal. Close the connection to the server, then reconnect.

```
:remote close
:remote connect tinkerpop.server conf/remote.yaml
:remote console
```

Now let's see if the two vertices we added are still remembered after the server restart.

```
gremlin> g.V().count()
==>2
```

The result shows that the vertices were remembered. This is the expected result because we configured JanusGraph to use a storage backend that persists the graph data.

You can explore the file system to see the folders that were created under the JanusGraph root folder to store the graph data and the mixed index data. The folder "db/berkeley" contains a few files. This is where the graph data (vertices, edges, properties, ...) is stored. These files are managed by Berkeley DB. The other folder "db/searchindex" is empty because we did not create any mixed indexes. But if we do, Lucene will write the mixed index data under this folder.

---

**Other articles in this series:**

1. [Installing JanusGraph and Testing it With the InMemory Storage Backend](../installing-janusgraph-and-testing-it-with-the-inmemory-storage-backend/index.md)
2. Configuring JanusGraph to Use Oracle Berkeley DB
