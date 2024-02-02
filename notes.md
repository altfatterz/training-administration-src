```bash
$ docker compose exec tools bash
$ cat <<EOF >~/.myvars
export CLUSTERID=Nk018hRAQFytWskYqtQduw \
export CONTROLLERS="9991@controller-1:19093,9992@controller-2:29093,9993@controller-3:39093"
export BOOTSTRAPS="kafka-1:9092,kafka-2:9092,kafka-3:9092"
EOF
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
```

```bash
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-1/00000000000000000000.index

offset: 35 position: 4329
offset: 65 position: 8440
offset: 98 position: 12599
offset: 129 position: 16819
offset: 161 position: 21161
offset: 192 position: 25394
offset: 225 position: 29518
offset: 254 position: 33799
offset: 286 position: 38032
offset: 315 position: 42265
offset: 348 position: 46472
```

```bash
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-1/00000000000000000000.timeindex

timestamp: 1706471402969 offset: 35
timestamp: 1706471402999 offset: 65
timestamp: 1706471403032 offset: 98
timestamp: 1706471403552 offset: 129
timestamp: 1706471403584 offset: 161
timestamp: 1706471403615 offset: 192
timestamp: 1706471403648 offset: 225
timestamp: 1706471403677 offset: 254
timestamp: 1706471403709 offset: 286
timestamp: 1706471403744 offset: 315
timestamp: 1706471403772 offset: 347
timestamp: 1706471404037 offset: 371
```

```bash
$ kafka-run-class kafka.tools.DumpLogSegments --files /var/lib/kafka/data/replicated-topic-1/00000000000000000372.snapshot

producerId: 1004 producerEpoch: 0 coordinatorEpoch: -1 currentTxnFirstOffset: OptionalLong.empty lastTimestamp: 1706471404037 firstSequence: 370 lastSequence: 371 lastOffset: 371 offsetDelta: 1 timestamp: 1706471404037
```

```bash
$ kafka-metadata-quorum --bootstrap-server $BOOTSTRAPS describe --status
```

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

References:

- How to optimize your Kafka producer for throughput
  https://developer.confluent.io/tutorials/optimize-producer-throughput/confluent.html