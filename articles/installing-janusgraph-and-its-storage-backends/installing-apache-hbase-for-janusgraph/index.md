## Installing JanusGraph and Its Storage Backends

# 4. Installing Apache HBase for JanusGraph

### Introduction

This is the fourth article in a series of articles about installing and configuring JanusGraph and its storage backends on a Linux Ubuntu server.

In this article, I will explain how to install and run the Apache HBase database. And how to configure JanusGraph to use it as its storage backend.

### Download and Extract HBase

At the time of writing this article, [JanusGraph's latest release page on GitHub](https://github.com/JanusGraph/janusgraph/releases/latest) says that JanusGraph is compatible with HBase 2.1.5. So I will get this version's download link from the [Apache HBase archive](https://archive.apache.org/dist/hbase/), and I will use the `wget` shell command to download it to my `/opt` directory. Then I will use the `tar` command to extract the contents of the downloaded archive.

```shell
cd /opt
wget https://archive.apache.org/dist/hbase/2.1.5/hbase-2.1.5-bin.tar.gz
tar -xf hbase-2.1.5-bin.tar.gz
```

### Create a Linux User for Running HBase

I think it is a good idea to create a Linux user for running the HBase server. This user can be given only the permissions needed by HBase and not more. This dedicated user will also make it easier to find the HBase server process if we need to monitor it or kill it.

```shell
adduser hbase
```

You will be prompted to enter the password for this new user.

Now let's make this newly created user the owner of the HBase root folder and all its contents.

```shell
chown -R hbase:hbase hbase-2.1.5
```

### Set the JAVA_HOME Environment Variable

We must set the JAVA_HOME environment variable before starting HBase. So we need to find the path to the Java Home directory. This path will be different depending on the Linux distribution and the installed JRE/JDK package.

I used the `which` command to find the location of the `java` command. The I used the `ll` command to get the details of this file. It turned out to be a symbolic link pointing to another symbolic link pointing to the actual "java" executable.

```
$ which java
/usr/bin/java
$ ll /usr/bin/java
lrwxrwxrwx 1 root root 22 Dec 15 17:22 /usr/bin/java -> /etc/alternatives/java*
$ ll /etc/alternatives/java
lrwxrwxrwx 1 root root 46 Dec 15 17:22 /etc/alternatives/java -> /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java*
$ ll /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
-rwxr-xr-x 1 root root 14632 Nov  8 19:38 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java*
```

So the path to the java executable is "/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java". And the Java Home folder is one level above the "bin" folder containing the "java" executable. So the Java Home folder is "/usr/lib/jvm/java-8-openjdk-amd64/jre/".
