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






References:

- How to optimize your Kafka producer for throughput
  https://developer.confluent.io/tutorials/optimize-producer-throughput/confluent.html