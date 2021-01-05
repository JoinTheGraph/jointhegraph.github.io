## Installing JanusGraph and Its Storage Backends

# 3. Installing Apache Cassandra for JanusGraph

### Introduction

This is the third article in a series of articles about installing and configuring JanusGraph and its storage backends on a Linux Ubuntu server.

In this article, I will explain how to install and run the Apache Cassandra database. And how to configure JanusGraph to use it as its storage backend.

### Download and Extract Cassandra

At the time of writing this article, [JanusGraph's latest release page on GitHub](https://github.com/JanusGraph/janusgraph/releases/latest) says that JanusGraph is compatible with Cassandra 3.11.0. So I will get this version's download link from the [Apache Cassandra archive](http://archive.apache.org/dist/cassandra/), and I will use the `wget` shell command to download it to my `/opt` directory. Then I will use the `tar` command to extract the contents of the downloaded archive.

```shell
cd /opt
wget http://archive.apache.org/dist/cassandra/3.11.0/apache-cassandra-3.11.0-bin.tar.gz
tar -xf apache-cassandra-3.11.0-bin.tar.gz
```

### Create a Linux User for Running Cassandra

I think it is a good idea to create a Linux user for running the Cassandra server. This user can be given only the permissions needed by Cassandra and not more. This dedicated user will also make it easier to find the Cassandra server process if we need to monitor it or kill it.

```shell
adduser cassandra
```

You will be prompted to enter the password for this new user.

Now let's make this newly created user the owner of the Cassandra root folder and all its contents.

```shell
chown -R cassandra:cassandra apache-cassandra-3.11.0
```

### Run the Cassandra Server

Switch to the "cassandra" user and run the "cassandra" shell script.

```shell
su cassandra
/opt/apache-cassandra-3.11.0/bin/cassandra -f
```

The `-f` flag runs the Cassandra server in the foreground so we can see the output messages and easily shutdown the server when we want by pressing CTRL + c

![Cassandra Server output](cassandra-server-output.png)

The screenshot above shows the Cassandra server output. The most important piece of information is the port number that Cassandra is listening to.

TODO: Complete article
