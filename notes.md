Kafka Options explorer: https://www.conduktor.io/kafka/kafka-options-explorer/

```bash
$ docker compose exec tools bash
```

```bash
$ cat <<EOF >~/.myvars
export CLUSTERID=Nk018hRAQFytWskYqtQduw
export CONTROLLERS="9991@controller-1:19093,9992@controller-2:29093,9993@controller-3:39093"
export BOOTSTRAPS="kafka-1:9092,kafka-2:9092,kafka-3:9092"
EOF
```

```bash
$ x='source ~/.myvars' ; grep -qxF "$x" ~/.bashrc || echo $x >> ~/.bashrc
$ source ~/.bashrc
```

You will have the variables set again in whenever you connect again:

```bash
$ docker compose exec tools bash
$ echo $CLUSTERID
$ echo $CONTROLLERS
$ echo $BOOTSTRAPS 
```

### Durability 

```bash
$ kafka-topics \
--create \
--bootstrap-server $BOOTSTRAPS \
--topic replicated-topic \
--partitions 6 \
--replication-factor 2 \
--config segment.bytes=50000
```

```bash
$ kafka-topics \
--describe \
--bootstrap-server $BOOTSTRAPS \
--topic replicated-topic
```

```bash
$ kafka-producer-perf-test \
--topic replicated-topic \
--num-records 6000 \
--record-size 100 \
--throughput 1000 \
--producer-props bootstrap.servers=$BOOTSTRAPS
```

```bash
$ grep replicated-topic /var/lib/kafka/data/recovery-point-offset-checkpoint
$ grep replicated-topic /var/lib/kafka/data/replication-offset-checkpoint
$ ls -l /var/lib/kafka/data/replicated-topic-0
$ cat /var/lib/kafka/data/replicated-topic-0/leader-epoch-checkpoint
```

```bash
$ kafka-dump-log --print-data-log --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.log
$ kafka-dump-log --print-data-log --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.index
$ kafka-dump-log --print-data-log --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.timeindex
$ kafka-dump-log --print-data-log --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.snapshot

$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.log
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.index
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.timeindex
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-0/00000000000000000000.snapshot
```

Produce more records:

```bash
$ kafka-producer-perf-test --topic replicated-topic --num-records 100000 --record-size 100 --throughput 1000 --producer-props bootstrap.servers=$BOOTSTRAPS
```

Check the value of `replication-offset-checkpoint` (high water mark) and `recovery-point-offset-checkpoint` (offset up to which is flushed to disk)

```bash
$ cat replication-offset-checkpoint | grep replicated-topic
$ cat recovery-point-offset-checkpoint | grep replicated-topic
```

```bash
$ kafka-dump-log --print-data-log --files /var/lib/kafka/data/__consumer_offsets-3/00000000000000000000.log
$ kafka-run-class kafka.tools.ConsumerOffsetChecker --topic replicated-topic
```

### kafka-consumer-groups

```bash
$ kafka-consumer-groups --bootstrap-server $BOOTSTRAPS --list
$ kafka-consumer-groups --bootstrap-server $BOOTSTRAPS --describe --group 
$ kafka-consumer-groups --bootstrap-server=$BOOTSTRAPS --describe --group console-consumer-42894
```

### kafka-metadata-quorum

```bash
$ kafka-metadata-quorum --bootstrap-server $BOOTSTRAPS describe --status
```

```bash
$ kafka-metadata-quorum --bootstrap-server kafka-1:9092 describe --replication

NodeId	LogEndOffset	Lag	LastFetchTimestamp	LastCaughtUpTimestamp	Status
9991  	2319        	0  	1706715517260     	1706715517260        	Leader
9992  	2319        	0  	1706715516927     	1706715516927        	Follower
9993  	2319        	0  	1706715516927     	1706715516927        	Follower
1     	2319        	0  	1706715516926     	1706715516926        	Observer
2     	2319        	0  	1706715516926     	1706715516926        	Observer
3     	2319        	0  	1706715516926     	1706715516926        	Observer
```

### kafka-metadata-shell

```bash
$ kafka-metadata-shell --cluster-id $CLUSTERID --controllers $CONTROLLERS
$ kafka-metadata-shell --cluster-id $CLUSTERID --controllers $CONTROLLERS ls image/cluster
$ kafka-metadata-shell --cluster-id $CLUSTERID --controllers $CONTROLLERS ls image/topics/byName
```

### kafka-broker-api-versions

```bash
$ kafka-broker-api-versions --bootstrap-server kafka-1:9092 --version
```

### kafka-transactions

```bash
$ kafka-transactions --bootstrap-server kafka-1:9092 list
TransactionalId              	Coordinator	ProducerId	TransactionState
connect-cluster-kafka-connect	2          	1010      	CompleteCommit
```

### kafka-rebalance-cluster

```bash
$ kafka-rebalance-cluster --bootstrap-server kafka-1:9092 --status
```


```bash
$ cat /etc/kafka/server.properties | grep log.dirs
log.dirs=/var/lib/kafka
```

```bash
$ kafka-consumer-groups --bootstrap-server=$BOOTSTRAPS --list
$ kafka-consumer-groups --bootstrap-server=$BOOTSTRAPS --describe --group console-consumer-42894
```

### __consumer_offsets

- has 50 partitions, replication factor 3

### __cluster_metadata

- has 1 partition

### Managing a Kafka cluster

```bash
$ cat /etc/kafka/kafka.properties

confluent.metrics.reporter.bootstrap.servers=kafka-1:9092,kafka-2:9092,kafka-3:9092
inter.broker.listener.name=DOCKER
jmx.port=10001
process.roles=broker
controller.listener.names=CONTROLLER
metric.reporters=io.confluent.metrics.reporter.ConfluentMetricsReporter
group.initial.rebalance.delay.ms=0
controller.quorum.voters=9991@controller-1:19093,9992@controller-2:29093,9993@controller-3:39093
jmx.hostname=localhost
node.id=1
advertised.listeners=DOCKER://kafka-1:9092, EXTERNAL://kafka-1:19092
listener.security.protocol.map=CONTROLLER:PLAINTEXT,DOCKER:PLAINTEXT,EXTERNAL:PLAINTEXT
broker.rack=rack-0
listeners=DOCKER://kafka-1:9092, EXTERNAL://kafka-1:19092
zookeeper.connect=
log.dirs=/var/lib/kafka/data
confluent.balancer.enable=true
```

```bash
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --describe
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --describe --all
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --describe --all | grep min.insync.replicas
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --describe --all | grep log.cleaner.threads
$ kafka-configs --bootstrap-server kafka-1:9092 --broker-defaults --alter --add-config log.cleaner.threads=2
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --alter --add-config log.cleaner.threads=3
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --alter --delete-config log.cleaner.threads
$ kafka-configs --bootstrap-server kafka-1:9092 --broker 1 --describe --all | grep log.cleaner.threads

$ kafka-topics --bootstrap-server kafka-1:9092 --create --topic my_topic --partitions 1 --replication-factor 3 --config segment.bytes=1000000
$ kafka-configs --bootstrap-server kafka-1:9092 --describe -topic my_topic
$ kafka-configs --bootstrap-server kafka-1:9092 --alter --topic my_topic --add-config segment.bytes=1000000
$ kafka-configs --bootstrap-server kafka-1:9092 --alter --topic my_topic --delete-config segment.bytes
$ kafka-configs --bootstrap-server kafka-1:9092 --describe -topic my_topic --all
$ kafka-topics --bootstrap-server kafka-1:9092 --describe --topic my_topic

$ kafka-topics --bootstrap-server kafka-1:9092 --delete --topic my_topic
```

Order of precedence:
- topic settings
- dynamic broker config
- dynamic cluster-wide config
- static server config in server.properties
- kafka default

```bash
$ kafka-topics --bootstrap-server $BOOTSTRAPS --create --topic test --partitions 3 --replication-factor 1
$ kafka-topics --bootstrap-server $BOOTSTRAPS --describe --topic test
$ cat << EOF > replicate_topic_test_plan.json
{"version":1,
"partitions":[
{"topic":"test","partition":0,"replicas":[1,2,3]},
{"topic":"test","partition":1,"replicas":[1,2,3]},
{"topic":"test","partition":2,"replicas":[2,3]}]
}
EOF
$ kafka-reassign-partitions --bootstrap-server $BOOTSTRAPS --reassignment-json-file replicate_topic_test_plan.json --verify
$ kafka-reassign-partitions --bootstrap-server $BOOTSTRAPS --reassignment-json-file replicate_topic_test_plan.json --execute
$ kafka-topics --bootstrap-server $BOOTSTRAPS --describe --topic test   
```

### Managing a Kafka cluster

```bash
$ kafka-topics --bootstrap-server $BOOTSTRAPS --create --topic moving --replica-assignment 1:2,2:1,1:2,2:1,1:2,2:1
$ kafka-topics --bootstrap-server $BOOTSTRAPS --describe --topic moving

$ confluent-rebalancer execute --bootstrap-server $BOOTSTRAPS --metrics-bootstrap-server $BOOTSTRAPS --throttle 1000000 --verbose
$ confluent-rebalancer status --bootstrap-server $BOOTSTRAPS

// show throttling configs
$ kafka-configs --bootstrap-server $BOOTSTRAPS --describe --topic moving
$ kafka-configs --bootstrap-server $BOOTSTRAPS --describe --entity-type brokers
```


### Consumer groups

```bash
$ kafka-topics --bootstrap-server $BOOTSTRAPS --create --topic grow-topic --partitions 6 --replication-factor 3
$ kafka-console-producer --bootstrap-server $BOOTSTRAPS --topic grow-topic
$ kafka-console-consumer --consumer-property group.id=test-consumer-group --from-beginning --topic grow-topic --bootstrap-server $BOOTSTRAPS
$ kafka-consumer-groups --bootstrap-server $BOOTSTRAPS --group test-consumer-group --describe

$ kafka-topics --bootstrap-server $BOOTSTRAPS --alter --topic grow-topic --partitions 12
$ kafka-topics --bootstrap-server $BOOTSTRAPS --describe --topic grow-topic
// wait 5 minutes
$ kafka-consumer-groups --bootstrap-server $BOOTSTRAPS --group test-consumer-group --describe
```

```bash
$ kafka-console-producer --bootstrap-server $BOOTSTRAPS --topic new-topic
$ kafka-console-consumer  --from-beginning --topic new-topic --group new-group --bootstrap-server $BOOTSTRAPS

$ kafka-console-consumer --topic __consumer_offsets --bootstrap-server $BOOTSTRAPS \
--formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" | grep new-topic

[new-group,new-topic,0]::OffsetAndMetadata(offset=3, leaderEpoch=Optional[0], metadata=, commitTimestamp=1707684378509, expireTimestamp=None)
```

### Secure cluster


```bash
$ openssl req -text -noout -verify -in kafka-1.csr

Subject: C=US, ST=Ca, L=PaloAlto, O=CONFLUENT, OU=TEST, CN=kafka-1
```

`ssl.endpoint.identification.algorithm` 
- The endpoint identification algorithm used by clients to validate server host name.
- Clients including client connections created by the broker for interbroker communication verify that the broker host name matches the host name in the broker’s certificate.
- Disable server host name verification by setting it to empty string


### Kafka Connect

```bash
$ docker-compose exec -u root kafka-connnect bash
$ confluent-hub install confluentinc/kafka-connect-jdbc:10.7.4
$ docker-compose restart kafka-connect
$ docker-compose logs kafka-connect | grep -i "INFO .* Finished starting connectors and tasks"

$ ls -l /usr/share/confluent-hub-components/
$ cat /etc/kafka-connect/kafka-connect.properties
```

```bash
$ insert into years(name,year) values('Anthony and Cleopatra',1606);
$ update years set name = 'Jane Doe' where id = 8;
```

```bash
$ curl -s -X POST \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Shakespeare-Sink",
    "config": {
      "topics": "shakespeare-years",
      "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
      "value.converter": "io.confluent.connect.avro.AvroConverter",
      "value.converter.schema.registry.url": "http://schema-registry:8081",
      "file": "/data/test.sink.txt"
    }
  }' http://kafka-connect:8083/connectors
```

```bash
$ curl http://kafka-connect:8083/connectors | jq .
```

### Producer performance

### What is the maximum throughput, no matter the latency?

(56.87 MB/sec) ??

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=600000 linger.ms=700 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 596302.921884 records/sec (56.87 MB/sec), 16.07 ms avg latency, 453.00 ms max latency, 13 ms 50th, 40 ms 95th, 50 ms 99th, 60 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 552760.819
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3498665.596
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 7.427
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 11.044
```

### What is the minimum latency, no matter the throughput?

18.78 ms avg latency ?

```bash
kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=16384 linger.ms=0 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 247279.920870 records/sec (23.58 MB/sec), 443.17 ms avg latency, 826.00 ms max latency, 454 ms 50th, 765 ms 95th, 793 ms 99th, 819 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 16218.913
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3258388.986
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 426.978
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 14.475
```

```bash
kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=200000 linger.ms=0 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 384319.754035 records/sec (36.65 MB/sec), 18.78 ms avg latency, 413.00 ms max latency, 14 ms 50th, 46 ms 95th, 55 ms 99th, 61 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 87929.413
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3404116.739
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 2.616
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 13.136
```


### What is the best balance of throughput and latency? (and defend your decision)




### Given a batch size of 100,000 Bytes, what linger time gives best performance? What do you notice? What do you wonder?
-- high throughput and low latency

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=100000 linger.ms=0 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 371471.025260 records/sec (35.43 MB/sec), 25.82 ms avg latency, 458.00 ms max latency, 21 ms 50th, 59 ms 95th, 74 ms 99th, 100 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 68366.532
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3400694.168
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 6.055
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 15.964
```

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=100000 linger.ms=100 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 457456.541629 records/sec (43.63 MB/sec), 19.35 ms avg latency, 412.00 ms max latency, 13 ms 50th, 55 ms 95th, 94 ms 99th, 109 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 95816.079
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3445442.055
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 5.577
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 13.222
```

bigger `linger.ms` decreases the throughput

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=100000 linger.ms=200 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 381533.765738 records/sec (36.39 MB/sec), 17.20 ms avg latency, 432.00 ms max latency, 10 ms 50th, 49 ms 95th, 62 ms 99th, 92 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 94743.494
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3403131.408
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 4.221
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 13.057
```


### Given a linger time of 500 ms, what batch size gives best performance? What do you notice? What do you wonder?
-- high throughput and low latency

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=16384 linger.ms=500 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 241429.261226 records/sec (23.02 MB/sec), 452.47 ms avg latency, 802.00 ms max latency, 544 ms 50th, 765 ms 95th, 789 ms 99th, 797 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 16254.801
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3253453.424
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 436.571
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 15.358
```

```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=400000 linger.ms=500 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"

1000000 records sent, 582072.176950 records/sec (55.51 MB/sec), 15.08 ms avg latency, 479.00 ms max latency, 10 ms 50th, 36 ms 95th, 59 ms 99th, 78 ms 99.9th.
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                   : 369128.010
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                            : 0.000
producer-metrics:outgoing-byte-rate:{client-id=perf-producer-client}                                               : 3496204.430
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                            : 5.624
producer-metrics:request-latency-avg:{client-id=perf-producer-client}                                              : 10.721
```





```bash
$ kafka-producer-perf-test \
--topic performance \
--num-records 1000000 \
--record-size 100 \
--throughput 10000000 \
--producer-props bootstrap.servers=$BOOTSTRAPS acks=all batch.size=400000 linger.ms=500 \
--print-metrics | grep -E "(^1000000 records sent|^producer-metrics:outgoing-byte-rate|^producer-metrics:bufferpool-wait-ratio|^producer-metrics:record-queue-time-avg|^producer-metrics:request-latency-avg|^producer-metrics:batch-size-avg)"
```


### Durability

- refers to the continued persistence of data without any data loss.

### Availability

-  the system uptime, i.e., the percentage of time the system is available and operational, allowing data to be written and read.

### Under replicated partitions

- This measurement, provided on each broker in a cluster, gives a count of the number of partitions for which the broker is the leader replica, where the follower replicas are not caught up. A high number may indicate a high load on the system.

### OfflinePartitionsCount:

- Number of partitions that are offline (which is not good for availability)


### Zero-Copy Transfer

A common data transfer from file to socket might go as follows:

1. The OS reads data from the disk into pagecache in the kernel space
2. The application reads the data from kernel space into a user-space buffer
3. The application writes the data back into kernel space into a socket buffer
4. The OS copies the data from the socket buffer to the NIC buffer, where it is sent over the network

However, if we have the same standardized format for data which doesn’t require modification, 
then we have no need for step 2 (copying the data from kernel space to user-space).

If we keep data in the same format as it will be sent over the network, 
then we can directly copy data from pagecache to NIC buffer.

### vmtouch - cannot install on Kafka node :(

```bash
$ $ apt-get update
$ $ apt-get install -y vmtouch
```


### Confluent for Kubernetes

https://docs.confluent.io/operator/current/overview.html

https://github.com/confluentinc/confluent-kubernetes-examples




References:

- How to optimize your Kafka producer for throughput
  https://developer.confluent.io/tutorials/optimize-producer-throughput/confluent.html