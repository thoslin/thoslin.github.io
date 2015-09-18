---
layout: post
title: "Cassandra: A Journey of Upgrade"
date: 2015-09-18 22:15
#comments: true
#categories: 
---
For the past few couple months, a huge burden on my shoulder had been upgrading our Cassandra cluster from 1.2.6 to 2.1. I've been investing a lot of working hours to figure out the solution. Now that it has been done, I feel it is worthwhile to write down the whole experience.
 
 
### Why Upgrade?

Actually the imperative reason is that we need transaction support in one of our services. And Cassandra 2.0 introduced a new feature called light-weight transaction, although it is light-weight, it somehow can fix our issue.

Besides that, there are also a couple of new features we can benefit from the upgrade:

- Improved native transport protocol. We're quite interested in the more stream requests over one connection. This is introduced in 2.1.
- Automatically paging support. We have queries for a large number of rows, this at a big chance will cause RPC timeout. Our workaround is to implements our own paging mechanism.
- Better counter. It is well known that Cassandra's distributed counter is buggy. They improve it in 2.1
 
 
### Infrastructure

- 10 nodes on production. 3 nodes on other stacks.
- Replication factor: 3
- Replication strategy: Simple strategy
- Consistency level: CL.ONE for both read and write.
 
 
### Upgrade Path

#### Driver upgrade

We're using a fairly old driver Cassandra called Pycassa, which is no longer maintained. And it is based on thrift protocol, which is deprecated/ditched in the version 3, so all the new and good stuff on the native protocol has nothing to do with Pycassa. Very naturally we switched to the recommended/official driver maintained by the Datastax.

Internally we don't have a layer for Cassandra, so refactoring is a lot of pain. We have to replace all the code usages of Pycassa among all services, and carefully update all unit tests.

We also bumped into some issues when deploying with the new driver: 
 

- [High CPU utilization when using asyncore event loop](https://datastax-oss.atlassian.net/browse/PYTHON-239). By now, this is not been fixed yet, so avoid using asycore, instead use gevent/libev.
- [Reconnect not initiated when all nodes are down](https://datastax-oss.atlassian.net/browse/PYTHON-364)
- [ConstantReconnectionPolicy does not work with max_attempts=None](https://datastax-oss.atlassian.net/browse/PYTHON-325)
- [Can't detect gevent monkey patch when using with uwsgi --gevent-monkey-patch option](https://datastax-oss.atlassian.net/browse/PYTHON-237). If you are monkey patch in uwsgi, use -gevent-early-monkey option.

The driver upgrade is not as smooth as I thought. A lot of back and forth happened and it took us almost two month or so to ship the upgrade.

#### No rolling upgrade?

Rolling upgrade should be a default option for a cluster upgrade. But unfortunately it is not supported between major versions of Cassandra. As it is documented [here](https://github.com/apache/cassandra/blob/trunk/NEWS.txt). We thought about workarounds. Like building a new Cluster and syncing data between two clusters. But building a new cluster is not our option due to some "policy", so we decided that we can tolerate some downtime, and that also means we will update each Cassandra instance in place.

#### Data backup and restore

It's important to have a backup of the data. In case something goes wrong, we can go back to the save point. When doing data backup, we demand that all services that access Cassandra should be stopped and keep data untouched during the process.

Below is a typical structure of one of  our Cassandra nodes:

/mnt/cassandra/

── commitlog_directory

── data_file_directories


"data_file_directories" is where Cassandra data files live, our goal is to backup this directory. We'll do a '[nodetool drain](http://docs.datastax.com/en/cassandra/2.0/cassandra/tools/toolsDrain.html)' on the node, which will flush all memtables to data files. After that We'll pack data_file_directories into one tarball and upload it to the cloud(to prevent disk failure of node). So we'll have two copies of data.


Procedure:

1. Drain the node
2. Clear snapshots
3. Shut down the node
4. Pack data files into a tarball.
5. Upload the tarball to swift


If something goes wrong and we want to abort the upgrade and go back to the old version. We simply retrieve the old data and unpack it to the Cassandra data file directory.

The backup and restore procedure are automated by Ansible scripts.

#### Upgrade

Upgrade directly from current version 1.2.6 to 2.1 is not possible. Since pre-2.0 SSTables are not supported by 2.1. A direct upgrade to 2.1, Cassandra would fail to start and following error would be raised:

```
java.lang.RuntimeException: Incompatible SSTable found. Current version ka is unable to read file: /var/lib/cassandra/data/system/schema_keyspaces/system-schema_keyspaces-ic-1. Please run upgradesstables.
        at org.apache.cassandra.db.ColumnFamilyStore.createColumnFamilyStore(ColumnFamilyStore.java:443) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.ColumnFamilyStore.createColumnFamilyStore(ColumnFamilyStore.java:420) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.Keyspace.initCf(Keyspace.java:327) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.Keyspace.<init>(Keyspace.java:280) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.Keyspace.open(Keyspace.java:122) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.Keyspace.open(Keyspace.java:99) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.db.SystemKeyspace.checkHealth(SystemKeyspace.java:558) ~[apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.service.CassandraDaemon.setup(CassandraDaemon.java:214) [apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.service.CassandraDaemon.activate(CassandraDaemon.java:443) [apache-cassandra-2.1.1.jar:2.1.1]
        at org.apache.cassandra.service.CassandraDaemon.main(CassandraDaemon.java:532) [apache-cassandra-2.1.1.jar:2.1.1]
```

So we upgraded to 2.0.0 and run upgradesstables command to upgrade SSTables. After that, we then upgrade from 2.0.0 to 2.1. 

Cassandra has an internal [version for SSTables](http://www.bajb.net/2013/03/cassandra-sstable-format-version-numbers/). During the upgrade, sstable version will be bumping from:

```
ic (1.2.6) --> ja (2.0.0) --> ka(2.1.3)
```


Procedure:

1. Install Cassandra 2.0.0. Before starting, set num_tokens to 1. The new version uses virtual nodes by default.
2. Upgrade SSTables
3. Drain and stop the node
4. Remove commit logs
5. Install Cassandra 2.1.3
6. Upgrade SSTables (Not necessary)
7. Bring back all services


The procedure looks simple and clear. While we had couple issue when doing test upgrade:

-  After upgrade from 1.2.6 to 2.0.0, Cassandra cannot start due to out of memory. It turns out when Cassandra starts, it would read the key cache. OutOfMemory when reading key cache seems to be a [known issue](http://cassandra-user-incubator-apache-org.3065146.n2.nabble.com/OOM-while-reading-key-cache-td7591267.html). The solution is to [clean up caches](http://mail-archives.apache.org/mod_mbox/cassandra-user/201307.mbox/%3C7B902431-9666-4D21-9324-8632BB6358F8@thelastpickle.com%3E ) before starting up.
- During the upgrade to 2.1, drain failed. It is a [bug](https://issues.apache.org/jira/browse/CASSANDRA-6374) and said to be fixed in 2.0.3. So we upgrade to 2.0.3 instead.

#### Data consistency

How to ensure data are not corrupted during the upgrade? I think this should be guaranteed by Cassandra. However when doing upgrade testing, we have a script to dump all Cassandra data before and after the upgrade to ensure data are not touched. This step is taken away when we're doing actual upgrade.

