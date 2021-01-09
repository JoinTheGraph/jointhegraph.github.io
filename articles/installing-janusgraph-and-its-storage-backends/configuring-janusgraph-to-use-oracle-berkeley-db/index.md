## Installing JanusGraph and Its Storage Backends

# 2. Configuring JanusGraph to Use Oracle Berkeley DB

<div style="text-align: center; margin-top: 2rem;"><iframe width="560" height="315" src="https://www.youtube.com/embed/KO_W6Ifh-5E" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>

### Introduction

This is the second article in a series of articles about installing and configuring JanusGraph and its storage backends on a Linux Ubuntu server.

In this article, I will explain how to configure JanusGraph to use "Oracle Berkeley DB Java Edition" as its storage backend.

### About Oracle Berkeley DB

Oracle Berkeley DB is an embedded key-value database library. The words "embedded" and "library" mean that Berkeley DB will be loaded and executed in the JanusGraph process as opposed to running in a separate process. So Berkeley DB is a database management library, not a stand-alone database server like Cassandra or HBase.

The Berkeley DB JAR file is included in the JanusGraph package that we downloaded in the previous article. The path to the JAR file is "lib/je-18.3.12.jar". So there is no need to download or install Berkeley DB separately.

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

### Critique of Berkeley DB

I am currently planning for an application that will use a graph database. And I will probably use JanusGraph unless I find a better alternative. I am still trying to decide on the storage backend that I will use with JanusGraph. My database/graph will be small enough to fit in one machine, so I do not think I will need horizontal scalability. So is Berkeley DB a good storage backend for my use case? I still do not know. But from my research so far, I found some pros and cons.

**The Pros**

1. Berkeley DB runs in the same process as JanusGraph. Other storage backends run as servers and listen to a port. So using Berkeley DB will save the overhead of communication between JanusGraph and the storage backend server.
2. Open source with a very permissive license (Apache 2.0). Note that I am talking about "Oracle Berkeley DB Java Edition". The other members of the Oracle Berkeley DB family have a much more restrictive license (AGPL 3). But I only care about the Java Edition because it is the only one that can be used with JanusGraph.

**The Cons**

Not future-proof at all. It seems like dead software. Even Oracle does not care very much about it. You can download Berkeley DB's source code from Oracle's website. But I could not find any public repository. There is a [discussion board](https://community.oracle.com/tech/developers/categories/berkeley_db_java_edition) for the product, but it is very inactive. The latest version downloadable from Oracle's website is 7.5.11. I have no idea when this version was released. Probably a long time ago. The version of the Berkeley DB library that comes with JanusGraph is 18.3.12. The JanusGraph developers got this JAR file from Maven Central.

This all seems messy to me. So I do not think I will use Berkeley DB in production unless it turns out to be much faster than Cassandra and HBase in my use case.

---

**Other articles in this series:**

1. [Installing JanusGraph and Testing it With the InMemory Storage Backend](../installing-janusgraph-and-testing-it-with-the-inmemory-storage-backend/index.md)
2. Configuring JanusGraph to Use Oracle Berkeley DB
3. [Installing Apache Cassandra for JanusGraph](../installing-apache-cassandra-for-janusgraph/index.md)
