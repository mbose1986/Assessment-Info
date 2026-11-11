**Assessment:**
**************************************************
1.1  ** Durability basics**

A producer writes with acks=all. Topic has RF=3 and min.insync.replicas=2.**
****
Answer:
•	What does acks=all actually wait for?
acks=all waits for acknowledgements from the leader and all in-sync replicas (ISRs) before considering a message successfully committed. It provides the strongest data durability and helps prevent data loss. However, it may introduce slightly higher latency compared to other acknowledgment settings because the producer must wait for all required replicas to acknowledge the message.

•	What happens when one broker holding a replica goes offline?
If the leader replica goes offline, one of the available in-sync follower replicas is automatically elected as the new leader, provided the min.insync.replicas requirement is satisfied. With a replication factor (RF) of 3 and min.insync.replicas=2, Kafka can continue operating with the remaining two in-sync replicas. Once the offline broker comes back online, its replica resumes functioning as a follower and catches up with the latest messages to become fully synchronized with the leader.
•	What happens when two brokers go offline?
With a replication factor (RF) of 3 and min.insync.replicas=2, Kafka requires at least two in-sync replicas to acknowledge writes. If 2 broker goes offline and no eligible in-sync replica is available, data may become temporarily unavailable. If unclean.leader.election.enable=true, Kafka may elect an out-of-sync replica as the new leader, allowing the partition to become available but potentially causing data loss or message-ordering inconsistencies. By default, this setting is false to prioritize data consistency and durability over availability.
•	Why is min.insync.replicas=2 (not 3) a common choice?

In most scenarios, setting min.insync.replicas=2 is a suitable approach because it ensures that at least two replicas remain in sync and available to acknowledge writes, providing a good balance between data durability and availability.

**1.2   Confluent Cloud on GCP — network path**

**You have a Confluent Cloud Dedicated cluster in us-central1 reached via Private Service Connect (PSC).**
Answer:
•	Draw the path from a GCE client in your VPC to the cluster (PSC endpoint, DNS resolution, bootstrap, per-broker connection).
       I worked on AWS , not have good idea particularly Google Cloud Platform (GCP) network path. But my answer based on my working experience in confluent cloud dedicated cluster on AWS. 
            Customer AWS/GCP VPC            
                                       |
    ┌──────────────────────────────┐   
     Client  │  Kafka Producer / Consumer  │   
                                       |
  │                DNS Resolution              │ Private DNS Zone
                                   │
  │   ┌──────────────▼───────────────┐   │
                                 Cloud DNS              
              Bootstrap / Broker Records 
                       Private IP Address      
                                       │
             AWS/ Google Private Network
                                         │
  ┌──────────────────▼───────────────────┐
  │       Confluent Cloud Network        │
  │                                      │
  │       PSC Service Attachment         │
  │                  │                   │
  │          ┌───────▼───────┐            │
  │          │ Kafka Cluster│            │
  │          │  us-central1 │            │
  │          │              │            │
  │          │ Broker 1     │            │
  │          │ Broker 2     │            │
  │          │ Broker 3     │            │
 
•	**Name the private DNS zone convention Confluent uses and which VPC it must be attached to.**
               Not sure about the answer, 
•	**Name one common misconfiguration and how you would detect it.**
One common misconfiguration is incorrect DNS mapping for PSC broker endpoints. Sometimes the bootstrap connection works, but individual broker connections fail because broker hostnames are not resolving to the correct private IP addresses.
Troubleshoot steps: 
1. Check Kafka client logs for DNS resolution failures, connection timeouts, or broker disconnection errors.
2. nc -zv <broker-hostname> 9092 to test network connectivity.

**1.3 Producer latency triage**
**Producer p99 latency jumped from 30 ms to ~400 ms. Throughput is unchanged. Broker CPU is flat.**
*****
Answer:
•	**Give 4 likely causes, ranked most-to-least likely.**
If this is producer side latency, then being a kafka dmin it is tough to get a conclusion , as this is client-side delay. But from kafka side I can check some of the parameters-------
Check with the network team, also check TCP connectivity using netstat----1
Broker disk I/O latency due to slow storage or increased disk contention-----2 [metric name=LogFlushRateAndTimeMs, disk utilization and availability using df-h]
Producer batching and acknowledgement delays due to linger.ms, acks=all, or slow follower replicas----3 [metric record-queue-time-avg, and record-queue-time-max]
•	**For each cause, name one metric or one log line you would check.**
[metric name=LogFlushRateAndTimeMs, disk utilization and availability using df-h]
              [metric record-queue-time-avg, and record-queue-time-max]
•	**On the broker host, what shell command would you use to check for recent GC pauses? Show the actual command.

tail -f /var/log/kafka/*gc*.log to monitor GC activity


1.4 **Ansible change gone wrong**
**Someone ran an Ansible play that restarted 3 of 6 brokers at the same time (serial: 3) to change a server.properties value. There were no health checks. After the play, several partitions show UnderReplicatedPartitions > 0.
******
Answer:
•	**Why is serial: 3 unsafe here?**
Because it allows to restart 3 brokers out of 6 brokers i.e. 50 % of the brokers simultaneously. it is not rolling restart which is recommendable. But here it is too risky as---1> replicas will be unavailable for some time as many leader replicas engages on those brokers, 2> as a result several partitions show UnderReplicatedPartitions > 0. Usually production cluster should restart brokers one-by-one in rolling restart manner.
•	**Name 3 safety checks the play should include.**
1>	Verify all Kafka brokers are online before starting the restart. (Grafana)
2>	Play should include dry-run and serial:1
3>	Verify all Kafka brokers are online after restart and broker health check too.

kafka-broker-api-versions.sh --bootstrap-server <host:port>
•	List the recovery steps in order.
=======================================================================
1.5 ZooKeeper vs KRaft — short check
Answer in 4–6 lines:
•	What role does ZooKeeper play in a Kafka cluster that uses it?
Zookeeper is used to manage the cluster metadata, broker coordination, controller selection. It is the heart of kafka cluster.
•	What replaces ZooKeeper in a KRaft cluster?
Kafka controller nodes quorum is used to replace zookeeper . It manages all zookeeper work inside kafka cluster and there are internal topics to store all metadata information. These are all theoretical answer as I did not work in kraft concept.
If you were building a new cluster today, which mode would you pick and why?
I would pick traditional zookeeper concept for production cluster as we are all known about the behavior of this pattern whereas I will be open for Kraft for POC to understand all concept, behavior, issue and then will test for non-prod cluster. As our main purpose is zero downtime, prevent data-loss for our client.







<img width="468" height="617" alt="image" src="https://github.com/user-attachments/assets/7f079979-e883-42f5-a3a2-240ec230788d" />
