# Research materials on Redis Cluster 

## Problem 1

### State

`- Active` 

- In this state, node actively handles requests. 

`- Passive` 

- In this state, node is in the standby state. Ready to replace the active node if something goes wrong.

### Mode 

`Active-Active clustering`

- An active-active cluster is typically made up of at least two nodes, both actively running the same kind of service simultaneously. The main purpose of an active-active cluster is to achieve load balancing. Load balancing distributes workloads across all nodes in order to prevent any single node from getting overloaded. Because there are more nodes available to serve, there will also be a marked improvement in throughput and response times.

- Performance : Allows use of all available performance. Higher performance than active / passive

- High availability (HA) : May be moderately responsive. In failure situation. If one node fails, the remaining nodes will assume the job for the failed node. But the nodes are also doing their own thing so taking more of the failed node's task may overload . Active-passive better than Active-active for high availability. 

- Reliability : Low reliability, although it still meets the availability, it's still better to have an excellent error than the active-passive. 

- Scalability : Can be easily extended horizontally and vertically.


`Active-Passive clustering` 

- An active-passive cluster also consists of at least two nodes. However, as the name "active-passive" implies, not all nodes are going to be active. In the case of two nodes, for example, if the first node is already active, the second node must be passive or on standby.

- Performance : This architecturally limited of total available performance. Must consume resources to maintain the standby node. More costly.  

- High availability (HA) : There is always a backup node for node failure should ensure the availability of the system.

- Reliability : High reliability. There is always redundancy in the event of an error with delay during failure is very low. 

- Scalability : There are a few limitations. Horizontally expandable. But it is still best to use vertical extension.

### Using case 

- Active-Active : If you want to optimize performance over cost, this is the structure to use. Can respond at a low level when the server is down (low HA). Depending on the needs of read / write, it is possible to increase the number of master or slave nodes.

- Active-passive : If you need system stability, high availability. 

## Problem 2

### Convert Standalone to Cluster 

#### `1.` I created a redis server in standalone mode. 

```
$ wget http://download.redis.io/releases/redis-6.0.6.tar.gz
$ tar xzf redis-6.0.6.tar.gz
$ cd redis-6.0.6
$ make
```

On `src` directory create redis.conf file

```
$ cd src
$ touch redis.conf
```

Configure the file as follows:  

```
port $port
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
dir ./
save 900 1
save 300 10
save 60 10000
```

Run redis standalone server with this command: 

```
./redis-server redis.conf
```

Create data test: 

```
for i in $(seq 1 10000);\
do ./redis-cli -p 7000 set foo$i hello$i;\
done
```

Now we have `dump.rdb` on src directory . Follow next step..

#### `2.` Create cluster with empty nodes, reshard all (empty) slots to just one instance, restart that instance with the dump in place and reshard again.

First you need a cluster. To start, create the config for 3 instances on port 7001-7003 (follow step 1 with $port 7001-7003)

Start all nodes and create cluster with 

```
./redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 --cluster-replicas 0
```

Let's reshard all slots to one instance

```
$ ./redis-cli --cluster reshard 127.0.0.1:7001 

>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: b978ede9f020161d929390879ba0f1e6028640fa 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master

M: 4ffe3048c376f407b9af90a5967d2c18b50c0db6 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master

M: a07d786e94074370ccdcc6b2d8316dda4020bace 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master

[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...

[OK] All 16384 slots covered.

How many slots do you want to move (from 1 to 16384)? 16384

What is the receiving node ID? b978ede9f020161d929390879ba0f1e6028640fa

Please enter all the source node IDs.

  Type 'all' to use all the nodes as source nodes for the hash slots.

  Type 'done' once you entered all the source nodes IDs.

Source node #1: all
```

Check slot and keys of nodes 

```
$ ./redis-cli --cluster check 127.0.0.1:7001
```

Copy `Dump.rdb` to 7001/redis-6.0.6/src directory. 

```
$ cp ../../../7000/redis-6.0.6/src/dump.rdb ./
```

Restart server

```
$ ./redis-cli -p 7001 shutdown nosave

$ ./redis-server redis.conf
```

#### `3.`  Now onto resharding to make use of all instances

Follow step 2 to resharding hash slot:

Reshard 5460 slots for node 7002
```
$ ./redis-cli --cluster reshard 127.0.0.1:7002 

>>> Performing Cluster Check (using node 127.0.0.1:7002)
M: 4ffe3048c376f407b9af90a5967d2c18b50c0db6 127.0.0.1:7002
   slots: (0 slots) master
M: a07d786e94074370ccdcc6b2d8316dda4020bace 127.0.0.1:7003
   slots: (0 slots) master
M: b978ede9f020161d929390879ba0f1e6028640fa 127.0.0.1:7001
   slots:[0-16383] (16384 slots) master

[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...

[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 5460

What is the receiving node ID? 4ffe3048c376f407b9af90a5967d2c18b50c0db6

Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```

Similar to 7003

```
$ ./redis-cli --cluster reshard 127.0.0.1:7003

>>> Performing Cluster Check (using node 127.0.0.1:7002)
M: 4ffe3048c376f407b9af90a5967d2c18b50c0db6 127.0.0.1:7002
   slots:[0-5459] (5460 slots) master
M: a07d786e94074370ccdcc6b2d8316dda4020bace 127.0.0.1:7003
   slots: (0 slots) master
M: b978ede9f020161d929390879ba0f1e6028640fa 127.0.0.1:7001
   slots:[5460-16383] (10924 slots) master

[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...

[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 5460

What is the receiving node ID? a07d786e94074370ccdcc6b2d8316dda4020bace

Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: b978ede9f020161d929390879ba0f1e6028640fa
Source node #2: done

Ready to move 5460 slots.
  Source nodes:
    M: b978ede9f020161d929390879ba0f1e6028640fa 127.0.0.1:7001
       slots:[5460-16383] (10924 slots) master
  Destination node:
    M: a07d786e94074370ccdcc6b2d8316dda4020bace 127.0.0.1:7003
       slots: (0 slots) master
```
When it finished some data should be moved to the second instance.Now we can check 

```
$ ./redis-cli --cluster check 127.0.0.1:7001

127.0.0.1:7001 (b978ede9...) -> 3329 keys | 5464 slots | 0 slaves.
127.0.0.1:7002 (4ffe3048...) -> 3343 keys | 5460 slots | 0 slaves.
127.0.0.1:7003 (a07d786e...) -> 3329 keys | 5460 slots | 0 slaves.
[OK] 10001 keys in 3 masters.
0.61 keys per slot on average.
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: b978ede9f020161d929390879ba0f1e6028640fa 127.0.0.1:7001
   slots:[10920-16383] (5464 slots) master
M: 4ffe3048c376f407b9af90a5967d2c18b50c0db6 127.0.0.1:7002
   slots:[0-5459] (5460 slots) master
M: a07d786e94074370ccdcc6b2d8316dda4020bace 127.0.0.1:7003
   slots:[5460-10919] (5460 slots) master
```

That looks good. We have all our data distributed onto 3 nodes, just as we wanted. Now we have server running on clustering mode , we can add slave node , very easy to scales horizontally or vertically.

## Problem 3

### I.  Persistance data types

`RDB :`

- Persistence performs point-in-time snapshots of your dataset at specified intervals.

- `Performance` : RDB allows faster restarts with big datasets compared to AOF.

`AOF :`

- Persistence logs every write operation received by the server, that will be played again at server startup, reconstructing the original dataset. Commands are logged using the same format as the Redis protocol itself, in an append-only fashion. Redis is able to rewrite the log in the background when it gets too big.

- `Performance` : AOF files are usually bigger than the equivalent RDB files for the same dataset. AOF can be slower than RDB depending on the exact fsync policy. In general with fsync set to every second performance is still very high, and with fsync disabled it should be exactly as fast as RDB even under high load. Still RDB is able to provide more guarantees about the maximum latency even in the case of an huge write load.

`Using case:`

- If your data is not so important and speed is a top priority then you should use RDB.
- If your data is very important you should use AOF or both. 

### II. Performance of Redis with persistance data types

`Benchmark test using redis-cluster` 

Tool test benchmark for redis-cluster. I'm setup cluster with 3 master node and 1 slave node for per master. Provide resources for each node: 100% one core, 1024 Mb RAM, limit disk IO speed 200mb/s (you can modify this number in config.sh file).

`Number of request: 2000000 , Key space: 100000, Pineline: 16` 

- Master and slave using AOF persistance 
- Master and slave using RDB persistance
- Master and slave using none persistance
- Master using RDB and slave using AOF persistance
- Master using RDB and slave using none persistance
- Master using AOF and slave using RDB persistance
- Master using AOF and slave using none persistance
- Master using none and slave using AOF persistance
- Master using none and slave using RDB persistance

 https://github.com/MinhWalker/redis-cluster-benchmark-test


#### Benchmark test using redis-sentinel 

`Benchmark test using redis-sentinel` 

- Redis Sentinel provides high availability for Redis. Sentinel is responsible for monitoring the status of master node, alerting and performing master node replacement when problems occur. So it gives performance equivalent to master-slave structure.


Tool test benchmark for redis-sentinel. I'm setup cluster with 1 master node, 2 slave node and 3 sentinel instances. Provide resources for each node: 100% one core, 1024 Mb RAM, limit disk IO speed 200mb/s (you can modify this number in config.sh file).

`Number of request: 2000000 , Key space: 100000, Pineline: 16` 

- Master and slave using AOF persistance 
- Master and slave using RDB persistance
- Master and slave using none persistance
- Master using RDB and slave using AOF persistance
- Master using RDB and slave using none persistance
- Master using AOF and slave using RDB persistance
- Master using AOF and slave using none persistance
- Master using none and slave using AOF persistance
- Master using none and slave using RDB persistance

https://github.com/MinhWalker/redis-sentinel-benchmark-test
