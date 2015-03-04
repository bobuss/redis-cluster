# Create a Redis Cluster with docker-machine

## Requirements

- [docker](https://github.com/docker/docker)
- [docker-machine](https://github.com/docker/machine)
- [redis-trib.rb](https://github.com/antirez/redis/blob/3.0/src/redis-trib.rb)

We use a home made redis-3.0 docker image (https://registry.hub.docker.com/u/bobuss/redis-cluster/), with the cluster mode enabled, and the classical ports exposed (6379 for redis and 16379 for the cluster).


## Spawn the instances

We are using the virtualbox driver here. We are going to spawn 3 instances. The cluster configuration will be 3 masters without slave (no failover).

```shell
for i in $(seq 1 3);\
do docker-machine create --driver virtualbox redis$i;\
done
```

which output:

```
INFO[0000] Creating SSH key...
INFO[0001] Creating VirtualBox VM...
INFO[0007] Starting VirtualBox VM...
INFO[0008] Waiting for VM to start...
INFO[0044] "redis1" has been created and is now the active machine.
INFO[0044] To point your Docker client at it, run this in your shell: $(docker-machine env redis1)
...
..
.
..
...
INFO[0000] Creating SSH key...
INFO[0001] Creating VirtualBox VM...
INFO[0008] Starting VirtualBox VM...
INFO[0008] Waiting for VM to start...
INFO[0044] "redis3" has been created and is now the active machine.
INFO[0044] To point your Docker client at it, run this in your shell: $(docker-machine env redis3)
```

## Install the redis-cluster docker image

```shell
for i in $(seq 1 3);\
do docker $(docker-machine config redis$i) pull bobuss/redis-cluster;\
done
```

which output:

```
Pulling repository bobuss/redis-cluster
dd52b1c716d8: Download complete
511136ea3c5a: Download complete
30d39e59ffe2: Download complete
...
..
.
..
...
225884a6d5de: Download complete
5138136a3bac: Download complete
61031df942a6: Download complete
7850a50f14ea: Download complete
Status: Downloaded newer image for bobuss/redis-cluster:latest
```

## Launch the redis on the instances

Do not forget the port 16379 (in fact, the redis port + 10000), which is used to make the cluster magic to happen.

```shell
for i in $(seq 1 3);\
do docker $(docker-machine config redis$i) run -d -p 6379:6379 -p 16379:16379 bobuss/redis-cluster;\
done
```

which output:

```
b5d5985a247092659ac70f24b154001ef4ccbd3887f077ce9323d04dc5f8b759
b22e0d9be1b072a8178815b71b9b834ede474e05241a96e65e54fc4b9eeeb19e
680076ffabd2e3bced95b5c9f3652963cbfaf4c0b6bed3e2359ef68f704d1c25
```

## Create the cluster

```shell
./redis-trib.rb create --replicas 0 \
$(docker-machine ip redis1):6379 \
$(docker-machine ip redis2):6379 \
$(docker-machine ip redis3):6379
```

which output:

```
>>> Creating cluster
Connecting to node 192.168.99.142:6379: OK
Connecting to node 192.168.99.143:6379: OK
Connecting to node 192.168.99.144:6379: OK
>>> Performing hash slots allocation on 3 nodes...
Using 3 masters:
192.168.99.142:6379
192.168.99.143:6379
192.168.99.144:6379
M: 344a19e87033ef62d5e8d31807156072e76221af 192.168.99.142:6379
   slots:0-5460 (5461 slots) master
M: 750c0fbe10c8eac1e7d89fdac8a8ff9826b78dbb 192.168.99.143:6379
   slots:5461-10922 (5462 slots) master
M: 4a17048efc59575749bcdb773a0316fa3c742444 192.168.99.144:6379
   slots:10923-16383 (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join.
>>> Performing Cluster Check (using node 192.168.99.142:6379)
M: 344a19e87033ef62d5e8d31807156072e76221af 192.168.99.142:6379
   slots:0-5460 (5461 slots) master
M: 750c0fbe10c8eac1e7d89fdac8a8ff9826b78dbb 192.168.99.143:6379
   slots:5461-10922 (5462 slots) master
M: 4a17048efc59575749bcdb773a0316fa3c742444 192.168.99.144:6379
   slots:10923-16383 (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```

## Check

Your 3 masters are redis1, redis2 and redis3.

```shell
$ redis-cli -h $(docker-machine ip redis1)
192.168.99.142:6379> cluster nodes
750c0fbe10c8eac1e7d89fdac8a8ff9826b78dbb 192.168.99.143:6379 master - 0 1425302527838 2 connected 5461-10922
4a17048efc59575749bcdb773a0316fa3c742444 192.168.99.144:6379 master - 0 1425302526819 3 connected 10923-16383
344a19e87033ef62d5e8d31807156072e76221af 172.17.0.3:6379 myself,master - 0 0 1 connected 0-5460

192.168.99.142:6379> cluster slots
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "192.168.99.143"
      2) (integer) 6379
2) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "192.168.99.144"
      2) (integer) 6379
3) 1) (integer) 0
   2) (integer) 5460
   3) 1) "172.17.0.3"
      2) (integer) 6379
```


## Add a node

We replay

- the instance creation,
- the docker image pull
- and the service launch

on a new instance called redis4.

```shell
$ docker-machine create --driver virtualbox redis4
INFO[0004] Creating SSH key...
INFO[0005] Creating VirtualBox VM...
INFO[0014] Starting VirtualBox VM...
INFO[0015] Waiting for VM to start...
INFO[0055] "redis4" has been created and is now the active machine.
INFO[0055] To point your Docker client at it, run this in your shell: $(docker-machine env redis4)

$ docker $(docker-machine config redis4) pull bobuss/redis-cluster
Pulling repository bobuss/redis-cluster
dd52b1c716d8: Pulling dependent layers
511136ea3c5a: Download complete
30d39e59ffe2: Downloading [======================================>            ] 28.67 MB/37.14 MB 3s
...
..
.
7850a50f14ea: Download complete
Status: Downloaded newer image for bobuss/redis-cluster:latest

$ docker $(docker-machine config redis4) run -d -p 6379:6379 -p 16379:16379 bobuss/redis-cluster
eaf2d20e43714eda631836d501b03be325e59f7fb835d47d54858f89b329936a

```

Then we add redis4 to the cluster.

```shell
$ ./redis-trib.rb add-node $(docker-machine ip redis4):6379 $(docker-machine ip redis1):6379
>>> Adding node 192.168.99.145:6379 to cluster 192.168.99.142:6379
Connecting to node 192.168.99.142:6379: OK
Connecting to node 192.168.99.143:6379: OK
Connecting to node 192.168.99.144:6379: OK
>>> Performing Cluster Check (using node 192.168.99.142:6379)
M: 344a19e87033ef62d5e8d31807156072e76221af 192.168.99.142:6379
   slots:0-5460 (5461 slots) master
   0 additional replica(s)
M: 750c0fbe10c8eac1e7d89fdac8a8ff9826b78dbb 192.168.99.143:6379
   slots:5461-10922 (5462 slots) master
   0 additional replica(s)
M: 4a17048efc59575749bcdb773a0316fa3c742444 192.168.99.144:6379
   slots:10923-16383 (5461 slots) master
   0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
Connecting to node 192.168.99.145:6379: OK
>>> Send CLUSTER MEET to node 192.168.99.145:6379 to make it join the cluster.
[OK] New node added correctly.

```

## Re-check our new 4 nodes cluster

```shell
$ redis-cli -h $(docker-machine ip redis1)
192.168.99.142:6379> cluster nodes
750c0fbe10c8eac1e7d89fdac8a8ff9826b78dbb 192.168.99.143:6379 master - 0 1425306745999 2 connected 5461-10922
e84d5b298edec6fb99cdb735f69edca91d97de75 192.168.99.145:6379 master - 0 1425306747014 0 connected
4a17048efc59575749bcdb773a0316fa3c742444 192.168.99.144:6379 master - 0 1425306747016 3 connected 10923-16383
344a19e87033ef62d5e8d31807156072e76221af 172.17.0.3:6379 myself,master - 0 0 1 connected 0-5460

```

