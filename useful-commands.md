```bash
$ kafka-broker-api-versions --bootstrap-server kafka-1:9092 --version

7.5.3-ce
```

```bash
$ kafka-metadata-quorum --bootstrap-server kafka-1:9092 describe --status

ClusterId:              Nk018hRAQFytWskYqtQduw
LeaderId:               9991
LeaderEpoch:            2
HighWatermark:          2307
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [9991,9992,9993]
CurrentObservers:       [1,2,3]
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


```bash
-- create a topic named `dummy-topic` and check the `__cluster_metadata` internal topic  
$ kafka-dump-log --cluster-metadata-decoder --files /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log | grep dummy-topic
```

```bash
$ kafka-metadata-shell --directory /var/lib/kafka/data/__cluster_metadata-0
```

```bash
$ kafka-transactions --bootstrap-server kafka-1:9092 list
TransactionalId              	Coordinator	ProducerId	TransactionState
connect-cluster-kafka-connect	2          	1010      	CompleteCommit
```

