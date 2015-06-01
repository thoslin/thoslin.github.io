---
layout: post
title: "Cassandra: Create a cluster on your local machine"
date: 2015-06-02 00:09
comments: false
---
This post will guide you through how to create a Cassandra cluster of multiple node on a local machine.

First, Let's grab a copy of Cassandra, I'm using a Ubuntu 12.04 box and gonna go with Cassandra 1.2.19.
```
mkdir test_cluster
cd test_cluster
wget http://www.us.apache.org/dist/cassandra/1.2.19/apache-cassandra-1.2.19-bin.tar.gz
tar xzf apache-cassandra-1.2.19-bin.tar.gz
```
The package already includes everything needed to start a Cassandra node. You can start it by sudo bin/cassandra. It will use all the defaults to start a Cassandra node. With data under /var/lib. with no initial token. We're not going to do that. As we're creating multiple nodes. Each node gonna will have its own directory and configurations. That said,
all nodes will share the same binaries comes within this tarball, but with different confs and directories for logs, data, commit logs.

We will build a directory structure like this:
```
node1
├── bin
├── commitlog
├── conf
├── data
├── logs
└── saved_caches
```
Now let's set up our first node:
```
cd test_cluster
mkdir node1
cd node1
mkdir commitlog data logs save_caches
cp -r ../apache-cassandra-1.2.19/bin .
cp -r ../apache-cassandra-1.2.19/conf .
```
We need to make some customizations before we bootstrap this node. And before we jump into that, we should generate initial_token for each node, unless you prefer virtual node, which is recommended. Anyway I use the following command to generate tokens:
```
python -c 'print [str(((2**64 / number_of_tokens) * i) - 2**63) for i in range(number_of_tokens)]'
```
We're going to create two nodes:
```
python -c 'print [str(((2**64 / 2) * i) - 2**63) for i in range(2)]'  
['-9223372036854775808', '0']
```
Now we can proceed with the setup:

####conf/cassandra.yaml 

The configuration file for Cassandra. There are couple items needs to be changed:
```
commitlog_directory: /home/tom/test_cluster/node1/commitlog
data_file_directories:
- /home/tom/test_cluster/node1/data
initial_token: -9223372036854775808
listen_address: 127.0.0.1
rpc_address: 127.0.0.1
saved_caches_directory: /home/tom/test_cluster/node1/saved_caches
```

####bin/cassandra.in.sh

The so-called include script. For seting environment variables needed by the start script bin/cassandra. We'll change following:
```
CASSANDRA_HOME=/home/tom/test_cluster/apache-cassandra-1.2.19
CASSANDRA_CONF=/home/tom/test_cluster/node1/conf
```
CASSANDRA_HOME is where the binaries live, CASSANDRA_CONF is where the conf for the node lives.

####conf/log4j-server.properies

```
log4j.appender.R.File=/home/tom/test_cluster/node1/logs/system.log
```
Now we're all set to start the node. Remember to set the CASSANDRA_INCLUDE to our cassandra.in.sh so that Cassandra will search the right place for confs.
```
CASSANDRA_INCLUDE=/home/tom/test_cluster/node1/bin/cassandra.in.sh /home/tom/test_cluster/node1/bin/cassandra -f
```
OK. If everything goes soothly, you should have the node up and running. Now Let's set up a second node. Nothing special. Just repeat the above steps. Use "node2" instead of "node1" when changing configurations. And use a different listen_address and rpc_address:
```
commitlog_directory: /home/tom/test_cluster/node2/commitlog
data_file_directories:
- /home/tom/test_cluster/node2/data
initial_token: 0
listen_address: 127.0.0.2
rpc_address: 127.0.0.2
saved_caches_directory: /home/tom/test_cluster/node2/saved_caches
```
And remember to make an extra modification to __conf/cassandra-env.sh__ to avoid port conflicts. Change default JMX_PORT to anything other than default 7199.
And start the node. The node will automatically join the ring.
```
~/test_cluster/node1 » bin/nodetool ring
Note: Ownership information does not include topology; for complete information, specify a keyspace
Datacenter: datacenter1
==========
Address    Rack        Status State   Load            Owns                Token                                      
                                                                          0                                          
127.0.0.1  rack1       Up     Normal  13.99 KB        50.00%              -9223372036854775808                      
127.0.0.2  rack1       Up     Normal  10.71 KB        50.00%              0              
```
Alright. You got a cluster running on your local machine!

Reference:
http://wiki.apache.org/cassandra/GettingStarted
