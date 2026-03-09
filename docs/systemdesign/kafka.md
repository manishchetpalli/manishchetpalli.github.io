## **Introduction**

How would you architect Kafka cluster for fault tolerance if you need to support 5 million messages per second?

--- 

## **Capacity Planning First**

- Message size (e.g., 1 KB vs 10 KB?)
- Retention period (e.g., 3 days, 7 days?)
- Replication factor (usually 3)
- Producer acks setting (acks=all for fault tolerance)

!!! Example
    If: 5M msg/sec & 1 KB per message

That’s:
    5M × 1 KB = 5 GB/sec incoming
    With RF=3 → 15 GB/sec internal write

So network + disk must handle that throughput.

--- 

## **Cluster Sizing**

For 5M msg/sec, a small cluster won’t work.

- Use 15–30 brokers minimum
- 10–25 Gbps network per broker
- NVMe SSDs (not HDD)
- Dedicated brokers (no shared infra)
- Kafka scales horizontally, so I would:
- Distribute partitions evenly
- Avoid hot brokers

--- 

## **Partition Strategy**

Throughput in Kafka scales with partitions.

Rule of thumb:
    One partition ≈ 5–15 MB/sec depending on hardware

To support 5 GB/sec:
    Need ~500–1000 partitions minimum across topics

!!! Example
    10 topics
    100 partitions each
    Replication factor = 3

Why high partitions:
    Parallelism for producers/consumers. Better load distribution

--- 

## **Replication & Fault Tolerance Design**

- Replication Factor:
    
    RF = 3 (minimum)
    RF = 4 if ultra-critical system

Min ISR: min.insync.replicas = 2

- Producer Setting:
    
    acks=all
    enable.idempotence=true
    retries=MAX

This ensures no data loss during broker failure and no duplicate writes.

--- 

## **Broker Configuration for Stability**

- Important configs:

    num.network.threads=8+

    num.io.threads=16+
    
    socket.send.buffer.bytes
    
    socket.receive.buffer.bytes
    
    log.segment.bytes (optimize)
    
    log.retention.hours
    
    unclean.leader.election.enable=false

Why disable unclean leader election? Prevents data loss.

--- 

## **Rack Awareness**

If running across availability zones: broker.rack=zone-a

Enable rack-aware replica placement.

So if one AZ fails: Cluster still runs and No data loss

--- 

## **Zookeeper vs KRaft**

If modern Kafka: Use KRaft mode

3 or 5 dedicated controllers

Separate from brokers in very large clusters

For 5M msg/sec: I would isolate controllers from brokers

--- 

## **Storage Design**

Multiple log.dirs per broker

RAID 0 (performance)

JBOD preferred (Kafka handles replication)

Monitor disk usage carefully

--- 

## **Consumer Scaling**

Consumers must match partition count.

If: 1000 partitions You can scale consumers up to 1000 in group

Avoid: Too many small partitions and Too few partitions (bottleneck)

--- 

## **Monitoring & Self-Healing**

- Critical tools:

    JMX metrics

    Prometheus + Grafana

    Cruise Control (for rebalancing)

- Alerting on:

    Under replicated partitions

    ISR shrink

    Network saturation

    Disk IO wait

--- 

## **Handling Broker Failure**

- If 1 broker dies: 

    Leader election happens

    ISR ensures no data loss

    Producers retry automatically

    No downtime (if RF ≥ 3)

    If entire AZ fails: Rack awareness ensures availability

--- 

## **Disaster Recovery**

- For enterprise setup:

    MirrorMaker 2
    
    Multi-cluster replication
    
    Active-passive or active-active setup