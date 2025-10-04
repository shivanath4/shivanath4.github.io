---
title: "How is Kafka so resilient?"
description: "About Kafka's durability and replication"
tag: Life
ogimage: "preview_tech_debt.png"

---

This blog is structured according to my curiosity. I start with high level abstractions and go low level if I think it is interesting or useful. If you do not find a particular path interesting, feel free to skip. I should mention that if you have no idea what Kafka is, this might not be a good place to start. Maybe see a couple of youtube videos and read ahead?

There are two main things in Kafka which is responsible for its resilience.
-  Replication (redundancy)
-  Durability (persistence)


Redundancy contribute to resilience by having the same eggs in multiple baskets. The durability part might not be as straight forward as to how it contributes to resilience, which led me to thinking a bit. I think making a system resilient fundamentally answers,  "if any point a part of a system goes down, how is your system going to handle it? can it remain functional?". With this thought, I made sense of durability. If a Kafka broker goes down, you'd still have the data as data is persisted to disk. You can make sense of the first two points in a similar fashion.

### Replication
If a Kafka topic's partition is 1 and replication is 3, we know that there are going to be 3 different brokers corrying the single partition's data. There will be 2 follower-brokers and 1 leader-broker. In case any one broker has an unexpected failure, we know that that we can fallback on the remaining 2 brokers for the data. Note that there is a leader for a partition. This helps in distributing traffic to all brokers as the leader is responsible for all reads and writes to the partition.

If a broker fails, will there be a downtime? Yes. For all the partitions where this specific broker is the leader, requests to read and write would be halted. Until a new leader is elected, it remains that way. For all other partitions, the In Sync Replicas (ISR) would be updated in their respective leader brokers. A copy of ISR is stored in Zookeeper and in the leader partition broker.

How does Kafka maintain consistency with replicas? A write request comes to the leader. It writes to disk and sends the message to all the replicas. It then updates ISR based on how many "acks" the leader gets back on successful write to replicas. Depending on `acks` value, you can define what a producer write success means for you - if it was written on leader or if it was written on all ISR replicas.

Talking about consistency, where does Kafka stand in CAP? Before answering the question, quick recap of what CAP theorem means. It says, in a distributed system, if there's a network failure (P), the system can only maintain consistency (C) or availability (A). CA examples don't really make sense as these would need to be not distributed. So Postgres and MySql fits. AP - Dynamo. CP - Zookeeper. While this is all armchair CS talk, [reality is much more nuanced](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html).

	Note that Consistency in CAP is different to Consistency in ACID. On a high level,
	C in CAP means every read gets the latest write.
	C in ACID means data moves from one valid state to another following all the constraints. 

To return back to answering the question, Kafka can be CP or AP depending on `unclean.leader.election.enable`, `min.insync.replicas`. Read more [here](https://kafka.apache.org/documentation/#design_ha).

Consider a scenario where all the replicas go down. Do we wait for a replica in ISR to come up or do we elect the first replica that comes up (that might not be in ISR) as leader? If `unclean.leader.election.enable` is set to true, the latter. If false, the former.

If `acks` is set to "all", the leader will [throw an exception](https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html) if it does write to atleast  `min.insync.replicas` number of replicas. If `min.insync.replicas` is greater, we are prioritising consistency over availability.

An offset is an identification number on a message in a partition. Helps in ordering and saving consumer offsets.
A high watermark is the latest offset which is successfully committed in all ISR replicas. Consumers see messages till this point. When `acks` is set to 0 or 1, it is possible that the leader has more data in the local log than the other brokers in ISR list and the possibility of ISR getting outdated. The other brokers send a Fetch Request on periodic intervals to fetch data from leader. In case the leader has more data that needs to be sent to other replicas but goes down without communicating the information, [there will be data loss](https://jack-vanlightly.com/blog/2018/9/14/how-to-lose-messages-on-a-kafka-cluster-part1). This does not happen when `acks` is set 'all' since the acknowledgement is sent to producer only after the leader gets acknowledgement back from all the in sync replicas. 

