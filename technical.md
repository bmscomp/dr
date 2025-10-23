# Kafka Multi-Site Deployment Architectures: Technical Guide

## Table of Contents
1. [Core Architecture Patterns](#core-architecture-patterns)
2. [Architecture Comparison](#architecture-comparison)
3. [Network Requirements](#network-requirements)
4. [Replication Strategies](#replication-strategies)
5. [Conflict Resolution](#conflict-resolution)
6. [Monitoring Strategy](#monitoring-strategy)
7. [Failover Procedures](#failover-procedures)
8. [Cloud Implementations](#cloud-implementations)

---

## Core Architecture Patterns

### 1. Stretch Cluster (Single Logical Cluster)

**Architecture:**
```
Site A (Primary DC)                    Site B (Secondary DC)
┌─────────────────────┐               ┌─────────────────────┐
│  Broker 1 (rack=A)  │               │  Broker 4 (rack=B)  │
│  Broker 2 (rack=A)  │◄─────────────►│  Broker 5 (rack=B)  │
│  Broker 3 (rack=A)  │  Sync Repl    │  Broker 6 (rack=B)  │
│                     │  <10ms RTT    │                     │
│  Controller 1       │               │  Controller 3       │
│  Controller 2       │               │                     │
└─────────────────────┘               └─────────────────────┘
         │                                     │
         └──────── Controller Quorum ──────────┘
```

**Technical Overview:**

Single Kafka cluster spanning two data centers using rack awareness for replica placement. Controllers (KRaft) or ZooKeeper ensemble form quorum across both sites. Synchronous replication with configurable `min.insync.replicas` enforces cross-site writes. Leader election and ISR management operate cluster-wide. Partition replicas distributed per `broker.rack` configuration using `RackAwareReplicaSelector`.

**Key Configurations:**
- `broker.rack`: Site identifier for rack-aware placement
- `min.insync.replicas=2`: Ensures cross-site acknowledgment
- `replica.lag.time.max.ms=30000`: Tolerates network latency
- `unclean.leader.election.enable=false`: Prevents data loss
- Controller quorum: 3 nodes minimum, distributed across sites

**Technical Characteristics:**
- Write path: Producer → Leader → ISR (cross-site) → Acknowledgment
- Network: Requires <10ms RTT, high bandwidth (throughput × RF × 1.5)
- Consistency: Linearizable reads/writes with proper acks configuration
- Failure handling: Automatic leader election, ISR shrinking/expanding

**Pros:**
- Single metadata store, unified cluster state
- Automatic cross-site replica placement via rack awareness
- Strong consistency with `acks=all` and proper `min.insync.replicas`
- No offset translation needed for consumers

**Cons:**
- Write latency = local latency + cross-site RTT + replication time
- Network partition causes ISR shrink, potential unavailability
- Limited to low-latency network environments (<10-15ms)
- Controller quorum requires cross-site consensus

**Use Cases:**
- Metro-area data centers with dedicated fiber
- Financial systems requiring strict consistency
- Single-region high-availability deployments
- Scenarios where network is predictable and low-latency

---

### 2. Active-Active with MirrorMaker 2.0

**Architecture:**
```
Site A Cluster                         Site B Cluster
┌─────────────────────┐               ┌─────────────────────┐
│  Cluster A          │               │  Cluster B          │
│  Independent        │               │  Independent        │
│  Broker 1-3         │               │  Broker 4-6         │
│                     │               │                     │
│  Topics:            │               │  Topics:            │
│  - orders           │               │  - orders           │
│  - site-b.orders ◄──┼───── MM2 ────┼──► site-a.orders    │
└─────────────────────┘  Async Repl  └─────────────────────┘
```

**Technical Overview:**

Two independent Kafka clusters with MirrorMaker 2.0 providing asynchronous bidirectional replication. MM2 runs as Kafka Connect distributed mode, using source connectors to consume from origin cluster and sink connectors to produce to target cluster. Each cluster maintains separate metadata, controller quorum, and offset management. Replication policies control topic naming and loop prevention.

**MM2 Components:**
- **MirrorSourceConnector**: Replicates topics from source to target
- **MirrorCheckpointConnector**: Syncs consumer group offsets
- **MirrorHeartbeatConnector**: Monitors replication health

**Key Configurations:**
- `replication.policy.class`: Controls topic naming (Identity vs Default)
- `sync.group.offsets.enabled=true`: Enables offset translation
- `emit.checkpoints.interval.seconds`: Checkpoint frequency
- `tasks.max`: Parallelism for replication tasks
- `topics.blacklist`: Prevents circular replication

**Technical Characteristics:**
- Replication lag: Typically seconds, depends on throughput and network
- Offset translation: Via checkpoints topic for consumer failover
- Loop prevention: Replication policy filters already-replicated topics
- Conflict handling: Application-level resolution required

**Pros:**
- Complete cluster independence, isolated failures
- Local write latency, no cross-site synchronous operations
- Flexible network requirements, tolerates high latency
- Survives network partitions gracefully

**Cons:**
- Eventual consistency model only
- Requires conflict resolution in applications
- Offset translation adds complexity to consumer failover
- Higher storage requirements (data duplicated)
- Complex monitoring across multiple clusters

**Use Cases:**
- Geographically distributed sites (>50ms RTT)
- Multi-region deployments with local traffic
- Scenarios tolerating eventual consistency
- Applications with natural data partitioning

---

### 3. Active-Passive (Primary-Backup)

**Architecture:**
```
Site A (Active)                        Site B (Passive)
┌─────────────────────┐               ┌─────────────────────┐
│  Primary Cluster    │               │  Standby Cluster    │
│  Broker 1-3         │───── MM2 ────►│  Broker 4-6         │
│                     │  One-way      │                     │
│  All Production     │               │  No Active Traffic  │
└─────────────────────┘               └─────────────────────┘
```

**Technical Overview:**

Single production cluster with unidirectional replication to standby cluster via MirrorMaker 2.0. Passive cluster remains idle or handles read-only queries. Consumer group offsets synchronized periodically to enable failover. Requires orchestration layer for failover automation (DNS updates, application reconfiguration).

**Key Configurations:**
- `primary->backup.enabled=true`, `backup->primary.enabled=false`
- `sync.group.offsets.enabled=true`: Critical for failover
- `sync.group.offsets.interval.seconds=60`: Balance RPO vs overhead
- `refresh.topics.interval.seconds`: Auto-discover new topics

**Technical Characteristics:**
- RPO: Determined by replication lag (typically seconds)
- RTO: Depends on failover automation (minutes to hours)
- Offset sync: Periodic, may have gaps during failover
- Failover: Requires DNS/load balancer updates, client reconfiguration

**Pros:**
- Simplest operational model, single source of truth
- No conflict resolution needed
- Lower cost (standby can be smaller capacity)
- Clear data lineage and ownership

**Cons:**
- Passive resources underutilized until failover
- Manual or semi-automated failover procedures
- Potential data loss during failover window (RPO > 0)
- No geographic optimization for reads

**Use Cases:**
- Traditional DR scenarios with clear primary site
- Cost-constrained deployments
- Applications intolerant to conflicts
- Compliance requiring single authoritative source

---

### 4. Active-Active with Aggregate Cluster

**Architecture:**
```
Site A Regional                        Site B Regional
┌─────────────────────┐               ┌─────────────────────┐
│  Regional Cluster   │               │  Regional Cluster   │
│  Broker 1-3         │               │  Broker 4-6         │
│  orders-site-a      │               │  orders-site-b      │
└──────────┬──────────┘               └──────────┬──────────┘
           │                                     │
           └────────────────┬────────────────────┘
                            │ MM2
                            ▼
                  Aggregate Cluster
                  ┌─────────────────────┐
                  │  Broker 7-9         │
                  │  orders-global      │
                  │  analytics          │
                  └─────────────────────┘
```

**Technical Overview:**

Three-tier architecture with regional operational clusters replicating to central aggregate cluster. Regional clusters handle low-latency operations, aggregate cluster handles analytics, cross-region queries, and global state. Requires careful topic design to avoid duplication and enable proper aggregation.

**Technical Characteristics:**
- Regional clusters: Optimized for low-latency operations
- Aggregate cluster: Optimized for storage and analytics
- Replication topology: Regional → Aggregate (possibly bidirectional)
- Data flow: Write locally, read locally or globally

**Key Design Patterns:**
- Topic naming: `{entity}-{region}` for regional, `{entity}-global` for aggregate
- Deduplication: Application-level or stream processing layer
- Retention: Short in regional (7d), long in aggregate (years)
- Hardware: SSD in regional, HDD acceptable in aggregate

**Pros:**
- Clear separation of operational vs analytical workloads
- Regional data sovereignty compliance
- Centralized global view for analytics
- Independent regional operations

**Cons:**
- Highest complexity, three clusters to manage
- Most expensive infrastructure footprint
- Additional MM2 instances and monitoring
- Potential data duplication if not designed carefully

**Use Cases:**
- Global enterprises with regional operations
- Heavy analytics requirements
- Data sovereignty compliance
- Multi-region with centralized reporting

---

### 5. Hub-and-Spoke

**Architecture:**
```
      Regional A              Regional B
      ┌─────────┐            ┌─────────┐
      │ Broker  │            │ Broker  │
      └────┬────┘            └────┬────┘
           │                      │
           └──────────┬───────────┘
                      │ MM2
                      ▼
              Central Hub
              ┌─────────────┐
              │  Broker 7-9 │
              │  Aggregated │
              └──────┬──────┘
                     │
          ┌──────────┴──────────┐
          │                     │
    Regional C            Regional D
    ┌─────────┐          ┌─────────┐
    │ Broker  │          │ Broker  │
    └─────────┘          └─────────┘
```

**Technical Overview:**

Star topology with central hub cluster and multiple spoke clusters. Each spoke replicates to hub (unidirectional or bidirectional). Hub aggregates data and optionally distributes to other spokes. Scales to dozens or hundreds of edge locations. Network requirements: spokes need connectivity to hub only, not to each other.

**Technical Characteristics:**
- Replication pattern: N spokes × 1 hub = N MM2 instances
- Hub capacity: Must handle aggregate throughput of all spokes
- Spoke isolation: Failures don't propagate between spokes
- Inter-spoke communication: Routes through hub (higher latency)

**Pros:**
- Scales to many edge locations
- Simplified network topology
- Centralized monitoring and control
- Clear data aggregation pattern

**Cons:**
- Hub becomes bottleneck and SPOF
- Hub requires high capacity
- Inter-spoke latency (two hops via hub)
- Complex hub failover scenarios

**Use Cases:**
- Retail chains with store clusters
- Manufacturing with factory edge clusters
- IoT deployments with edge processing
- Enterprise with many branch offices

---

### 6. Observer Cluster Pattern

**Architecture:**
```
Production Clusters                    Observer Cluster
┌─────────────────────┐               ┌─────────────────────┐
│  Site A Production  │               │  Read-Only Cluster  │
│  Broker 1-3         │───── MM2 ────►│  Broker 7-9         │
└─────────────────────┘               │                     │
                                      │  Analytics/Dev/Test │
┌─────────────────────┐               │  ML Pipelines       │
│  Site B Production  │───── MM2 ────►│  Compliance/Audit   │
│  Broker 4-6         │               └─────────────────────┘
└─────────────────────┘
```

**Technical Overview:**

Dedicated read-only cluster receiving replicated data from production clusters. Isolates non-production workloads (analytics, testing, development) from production performance impact. Typically uses cheaper hardware, longer retention, and relaxed performance SLAs.

**Technical Characteristics:**
- Replication: One-way from production to observer
- Lag tolerance: Seconds to minutes acceptable
- Hardware: Cost-optimized (HDD, smaller instances)
- Retention: Extended compared to production

**Pros:**
- Zero impact on production performance
- Safe environment for experimentation
- Cost-effective analytics platform
- Extended retention for compliance

**Cons:**
- Additional infrastructure costs
- Replication lag (stale data)
- Read-only limits testing scenarios
- Another cluster to operate

**Use Cases:**
- Analytics workload isolation
- Development and testing environments
- Compliance and audit trails
- ML model training

---

### 7. Tiered Storage with DR

**Architecture:**
```
Site A                                 Site B
┌─────────────────────┐               ┌─────────────────────┐
│  Hot: SSD (7d)      │               │  Hot: SSD (7d)      │
│  Broker 1-3         │───── MM2 ────►│  Broker 4-6         │
│  ↓ Tier             │               │  ↓ Tier             │
│  Cold: S3 (2y)      │── Replicate ─►│  Cold: S3 (2y)      │
└─────────────────────┘               └─────────────────────┘
```

**Technical Overview:**

Leverages Kafka tiered storage (3.0+) to separate hot data (local SSD) from cold data (cloud object storage). Hot data replicates via MM2, cold data replicates via cloud storage mechanisms. Reduces DR bandwidth requirements and broker storage costs while maintaining historical data access.

**Key Configurations:**
- `remote.log.storage.system.enable=true`
- `remote.log.manager.task.interval.ms`: Tiering frequency
- `local.retention.ms`: Hot tier retention
- `retention.ms`: Total retention (hot + cold)

**Technical Characteristics:**
- Hot tier: Recent data on broker local storage
- Cold tier: Historical data in S3/Azure Blob/GCS
- Tiering: Automatic based on segment age
- DR replication: Only hot tier via MM2, cold tier via cloud replication

**Pros:**
- Dramatic storage cost reduction
- Reduced DR bandwidth (only hot data)
- Leverages cloud storage durability
- Faster failover (less data to sync)

**Cons:**
- Higher latency for historical queries
- Feature maturity varies by Kafka version
- Cloud egress costs for reads
- Additional monitoring complexity

**Use Cases:**
- Long retention requirements (months/years)
- Cost-sensitive large-scale deployments
- Cloud-native architectures
- Compliance-driven retention

---

## Architecture Comparison

### Comparison Matrix

| Feature | Stretch Cluster | Active-Active | Active-Passive | Aggregate | Hub-Spoke | Observer | Tiered |
|---------|----------------|---------------|----------------|-----------|-----------|----------|--------|
| **Consistency** | Strong | Eventual | Strong* | Eventual | Eventual | Eventual | Varies |
| **Write Latency** | High (10-30ms) | Low (<5ms) | Low (<5ms) | Low (<5ms) | Low (<5ms) | N/A | Low (<5ms) |
| **Clusters** | 1 | 2 | 2 | 3+ | N+1 | N+1 | 2 |
| **Complexity** | Low | Medium | Low | High | Medium | Low | Medium |
| **Cost** | Medium | High | Low | Very High | High | Medium | Low |
| **Network** | <10ms | Relaxed | Relaxed | Relaxed | Relaxed | Relaxed | Relaxed |
| **Max Distance** | <100mi | Global | Global | Global | Global | Global | Global |
| **Conflicts** | None | Required | None | Required | Possible | N/A | Varies |
| **Failover** | Automatic | Manual | Manual | Complex | Complex | N/A | Manual |
| **RPO** | ~0 | Seconds | Seconds | Seconds | Seconds | N/A | Seconds |
| **RTO** | ~0 | Minutes | Minutes | Minutes | Minutes | N/A | Minutes |

*After failover completes

---

## Network Requirements

### Latency Requirements

**Stretch Cluster:**
- RTT: <10ms (ideal <5ms)
- Jitter: <2ms
- Packet loss: <0.01%
- Reason: Synchronous replication in write path

**Other Architectures:**
- RTT: <100ms acceptable (higher possible)
- Jitter: <10ms
- Packet loss: <0.1%
- Reason: Asynchronous replication via MM2

### Bandwidth Calculation

**Stretch Cluster:**
```
Required BW = Write Throughput × RF × Overhead Factor
Example: 100 MB/s × 3 × 1.5 = 450 MB/s = 3.6 Gbps
```

**Active-Active:**
```
Required BW = Write Throughput × Overhead Factor
Example: 100 MB/s × 1.2 = 120 MB/s = 960 Mbps
```

**Overhead factors:**
- Protocol overhead: 1.1-1.2x
- Retransmissions: 1.05-1.1x
- Metadata/heartbeats: 1.05x
- Combined: 1.2-1.5x

### Network Architecture

**Stretch Cluster Requirements:**
- Dedicated links (MPLS, dark fiber, or dedicated circuits)
- No shared internet paths
- Redundant physical paths
- QoS configuration prioritizing Kafka traffic

**MM2-Based Requirements:**
- Internet or VPN acceptable
- Shared WAN links usable
- Rate limiting to prevent saturation
- Monitoring for congestion

---

## Replication Strategies

### Stretch Cluster Replication

**Mechanism:** Built-in Kafka replication with rack awareness

**Configuration Strategy:**
```
Partition replicas placed across racks:
- Partition 0: [Broker1-SiteA, Broker2-SiteA, Broker4-SiteB]
- Partition 1: [Broker2-SiteA, Broker3-SiteA, Broker5-SiteB]
- Partition 2: [Broker3-SiteA, Broker4-SiteB, Broker6-SiteB]

min.insync.replicas=2 ensures at least one cross-site replica
```

**Failure Scenarios:**
- One broker fails: Leader election, ISR adjustment
- One site fails: Partitions with leaders in failed site elect new leaders
- Network partition: ISR shrinks, partitions may become unavailable if <min.insync.replicas

### MirrorMaker 2.0 Replication

**Architecture Components:**
- Source connectors: Consume from origin cluster
- Sink connectors: Produce to destination cluster
- Checkpoint connectors: Sync consumer offsets
- Heartbeat connectors: Monitor replication health

**Replication Flow:**
```
Origin Cluster → MM2 Consumer → MM2 Producer → Target Cluster
                      ↓
                 Checkpoints stored in target cluster
```

**Topic Naming Strategies:**

**DefaultReplicationPolicy:**
- Creates topics as `{source}.{topic}`
- Example: Site A topic `orders` → Site B topic `site-a.orders`
- Pros: Clear origin tracking
- Cons: Topic name changes

**IdentityReplicationPolicy:**
- Keeps original topic names
- Example: Site A topic `orders` → Site B topic `orders`
- Pros: No application changes
- Cons: Must prevent circular replication via blacklists

### Offset Synchronization

**Mechanism:**
MM2 checkpoint connector periodically reads consumer group offsets from source cluster and translates them to target cluster offsets using internal mapping tables.

**Checkpoint Topics:**
- `{source}.checkpoints.internal`: Stores offset mappings
- Updated every `emit.checkpoints.interval.seconds`

**Translation Process:**
1. Consumer commits offset 1000 in source cluster
2. MM2 maps source offset to target offset (e.g., 1005 due to replication timing)
3. Checkpoint written to target cluster
4. During failover, consumer reads checkpoint and resumes at 1005

**Limitations:**
- Approximate translation (not exact)
- Checkpoint lag means potential duplication or gaps
- Idempotent consumers recommended

---

## Conflict Resolution

### Conflict Types

**Type 1: Concurrent Writes**
Same key updated independently in both sites before replication propagates.

**Type 2: Causality Violations**
Write A depends on write B, but replication ordering differs between sites.

**Type 3: Delete-Update Conflicts**
One site deletes while other site updates same key.

### Resolution Strategies

#### 1. Last-Write-Wins (Timestamp-Based)

**Mechanism:** Use message timestamp or application timestamp to determine winner.

**Pros:**
- Simple to implement
- Low overhead
- Works for many use cases

**Cons:**
- Clock skew can cause incorrect ordering
- Loses concurrent updates
- Not suitable for collaborative editing

**Implementation:**
Track highest timestamp per key in local state store. Accept only messages with timestamps greater than stored value.

#### 2. Site Priority

**Mechanism:** Designate primary site; always prefer its writes over secondary.

**Pros:**
- Deterministic
- Simple logic
- Works when one site is clearly authoritative

**Cons:**
- Unfair to secondary site users
- Secondary site writes may be lost
- Doesn't handle multi-site equal scenarios

**Implementation:**
Include site identifier in message metadata. Consumer checks site priority and processes accordingly.

#### 3. Vector Clocks

**Mechanism:** Track causality using vector of counters per site.

**Pros:**
- Accurately detects causality and concurrency
- No clock synchronization required
- Handles complex scenarios

**Cons:**
- Higher overhead
- Requires client-side vector clock management
- Concurrent conflicts still need business logic resolution

**Implementation:**
Each write increments its site's counter. Compare vector clocks: if all counters in A ≤ B, then A happened-before B.

#### 4. CRDTs (Conflict-Free Replicated Data Types)

**Mechanism:** Use data structures with commutative, associative merge operations.

**Types:**
- **G-Counter**: Grow-only counter (increment-only)
- **PN-Counter**: Positive-negative counter (increment/decrement)
- **LWW-Element-Set**: Last-write-wins set
- **OR-Set**: Observed-remove set

**Pros:**
- Mathematically guaranteed eventual consistency
- No conflict resolution logic needed
- Handles concurrent operations correctly

**Cons:**
- Limited data types
- Higher storage overhead
- Requires application redesign

**Use Cases:**
- Inventory counts (PN-Counter)
- Shopping carts (OR-Set)
- Collaborative editing (specialized CRDTs)

---

## Monitoring Strategy

### Key Metrics

#### Cluster Health Metrics

**UnderReplicatedPartitions:**
- Critical metric, should always be 0
- Indicates partitions without sufficient replicas
- Causes: Broker failures, network issues, disk problems

**ISR Shrink/Expand Rate:**
- Monitors replica synchronization stability
- High shrink rate indicates network or broker problems
- Should be near-zero in healthy cluster

**OfflinePartitions:**
- Partitions with no available leader
- Critical availability issue
- Requires immediate attention

**Controller Active:**
- Only one controller should be active
- Multiple active = split brain
- Zero active = cluster unavailable

#### Replication Metrics

**Replica Lag (Stretch Cluster):**
- Measured in messages or bytes
- All replicas should be in-sync (lag = 0)
- Persistent lag indicates slow followers

**MM2 Replication Lag (Active-Active):**
- Measured as record count difference
- Acceptable lag varies by use case (seconds to minutes)
- Monitor trend, not absolute value

**Checkpoint Lag:**
- Time since last consumer offset checkpoint
- High lag means poor failover RPO
- Monitor per consumer group

#### Performance Metrics

**Request Latency (p99, p999):**
- Produce request latency
- Fetch request latency
- Should be consistent and within SLA

**Throughput:**
- Messages per second in/out
- Bytes per second in/out
- Per topic and cluster-wide

**Network Saturation:**
- Broker network utilization
- Should have headroom (< 70%)
- Monitor for sustained saturation

### Monitoring Stack

**Components:**
- **JMX Exporter**: Expose Kafka metrics to Prometheus
- **Prometheus**: Time-series database for metrics
- **Grafana**: Visualization and dashboards
- **Alertmanager**: Alert routing and notification

**Alert Severity Levels:**

**Critical (Page immediately):**
- UnderReplicatedPartitions > 0 for 5min
- OfflinePartitions > 0 for 1min
- Broker down for 1min
- MM2 worker down for 2min

**Warning (Notify, investigate):**
- ISR shrink rate elevated
- Replication lag increasing
- Consumer lag high (>threshold)
- Disk utilization >80%

**Info (Track, no immediate action):**
- Normal operational events
- Configuration changes
- Scheduled maintenance

---

## Failover Procedures

### Stretch Cluster Failover

**Scenario: Complete Site A failure**

**Automatic Behavior:**
1. Brokers in Site B detect Site A broker failures
2. Controller triggers leader election for affected partitions
3. Replicas in Site B promoted to leaders
4. ISR updated to exclude failed Site A brokers
5. Cluster continues operating with Site B only

**Operator Actions:**
1. Verify cluster health: Check UnderReplicatedPartitions
2. Monitor client connections: Ensure clients connected to Site B brokers
3. Assess data consistency: Verify min.insync.replicas satisfied before failure
4. Plan recovery: Prepare to bring Site A back online

**Recovery Process:**
1. Bring Site A brokers online
2. Brokers rejoin cluster and start catching up
3. Replicas sync from leaders in Site B
4. Once caught up, rejoin ISR
5. Optional: Rebalance leaders across sites

**Risks:**
- If failure occurred during writes with insufficient ISRs, data loss possible
- Network partition vs total failure: Different handling required
- Split-brain: Prevented by controller quorum

### Active-Active Failover

**Scenario: Site A complete failure**

**MM2 Behavior:**
1. Site B cluster continues operating normally
2. MM2 workers lose connection to Site A (source)
3. Replication from Site A to Site B stops
4. Replication from Site B to Site A continues trying (will fail)

**Application Failover:**

**Producers:**
1. Detect connection failure to Site A
2. Update bootstrap servers to Site B
3. Resume producing to Site B
4. May need to retry failed messages

**Consumers:**
1. Detect connection failure to Site A
2. Update bootstrap servers to Site B
3. Lookup translated offsets from checkpoint topic
4. Resume consumption from translated offset
5. May see some duplicates (idempotent processing required)

**Operator Actions:**
1. Confirm Site A unavailability
2. Update DNS/load balancer to route to Site B
3. Monitor Site B for capacity (now handling all traffic)
4. Disable MM2 workers trying to reach Site A
5. Notify applications of failover

**Recovery (Site A restored):**
1. Bring Site A cluster online
2. MM2 begins replicating from Site B to Site A
3. Site A catches up (catchup time depends on outage duration)
4. Once caught up, resume bidirectional replication
5. Evaluate conflict resolution for writes during outage
6. Optionally failback traffic to Site A

### Active-Passive Failover

**Scenario: Site A (active) failure**

**Preparation:**
- Site B (passive) already synchronized via MM2
- Consumer group offsets periodically checkpointed
- Runbooks prepared with failover steps

**Failover Steps:**
1. **Declare disaster**: Confirm Site A unrecoverable in acceptable timeframe
2. **DNS cutover**: Update DNS to point to Site B bootstrap servers
3. **Application reconfiguration**: Update client configs if not using DNS
4. **Verify consumer offsets**: Check checkpoint lag, assess data loss
5. **Enable Site B**: Scale up if needed, enable production traffic
6. **Monitor closely**: Watch for issues with offset translation, capacity

**Validation:**
1. Test producer writes to Site B
2. Test consumer reads from Site B
3. Verify consumer groups resumed correctly
4. Check for missing or duplicate messages
5. Monitor system health metrics

**Recovery (Site A restored):**
1. **Assess state**: Determine if Site A data current or stale
2. **Decision point**: 
   - If Site A data stale: Wipe and resync from Site B (now active)
   - If Site A data current: Evaluate data divergence
3. **Reverse replication**: Configure MM2 from Site B (active) to Site A (passive)
4. **Sync and verify**: Wait for full synchronization
5. **Failback (optional)**: Reverse failover process to restore Site A as active

---

## Cloud Implementations

### AWS Implementation

#### Stretch Cluster Architecture

**Network:**
- Two Availability Zones in same region
- VPC peering or Transit Gateway for cross-AZ communication
- Placement groups for low-latency networking

**Compute:**
- EC2 instances: m5.2xlarge or m6i.2xlarge
- EBS volumes: gp3 or io2 for low latency
- Enhanced networking enabled

**Configuration:**
- `broker.rack` = Availability Zone ID
- Security groups allowing all Kafka ports between AZs
- NLB for client connections

#### Active-Active Architecture

**Multi-Region Setup:**
- VPC in each region (us-east-1, eu-west-1)
- VPC Peering or AWS Transit Gateway for inter-region
- Route53 for DNS-based failover

**MSK (Managed Streaming for Kafka):**
- MSK cluster in each region
- Separate VPCs, separate configurations
- MM2 on EC2 or ECS for replication

**MirrorMaker 2.0 Deployment:**
- EC2 Auto Scaling group or ECS Fargate
- Distributed across multiple AZs
- CloudWatch metrics integration

**Cost Optimization:**
- Use VPC endpoints to avoid NAT gateway costs
- Leverage S3 for tiered storage
- Reserved instances for predictable workloads

### Azure Implementation

#### Stretch Cluster Architecture

**Network:**
- Two Availability Zones in same region
- VNet with subnets per AZ
- Network Security Groups for traffic control

**Compute:**
- VM: Standard_D4s_v3 or Standard_F8s_v2
- Premium SSD for storage
- Accelerated networking enabled

**Configuration:**
- `broker.rack` = Availability Zone
- Load Balancer for client access
- Azure Monitor for metrics

#### Active-Active Architecture

**Multi-Region Setup:**
- VNets in each region (East US, West Europe)
- VNet Peering for inter-region connectivity
- Traffic Manager for DNS-based routing

**Event Hubs (Kafka-compatible):**
- Event Hubs namespace per region
- Native Azure integration
- Consider limitations vs self-managed Kafka

**MM2 Deployment:**
- AKS (Azure Kubernetes Service) with Kafka Connect
- Container Instances for simpler deployments
- Application Insights for monitoring

**Cost Optimization:**
- Use Azure Spot VMs for non-critical workloads
- Leverage Azure Blob Storage for tiered storage
- Zone redundancy only where needed

### GCP Implementation

#### Stretch Cluster Architecture

**Network:**
- Two zones in same region
- VPC with subnet per zone
- Firewall rules for Kafka traffic

**Compute:**
- n2-standard-8 or c2-standard-8 instances
- Persistent SSD disks
- gVNIC for network performance

**Configuration:**
- `broker.rack` = Zone name
- TCP/UDP Load Balancer for clients
- Cloud Monitoring integration

#### Active-Active Architecture

**Multi-Region Setup:**
- VPC per region with global routing
- Cloud Interconnect or VPN for cross-region
- Cloud DNS for traffic management

**Deployment:**
- GKE for containerized Kafka/MM2
- Compute Engine for VM-based deployment
- Cloud Storage for tiered storage

**Monitoring:**
- Cloud Monitoring with custom metrics
- Cloud Logging for audit trails
- Alerting policies for critical events

### Kubernetes Deployment (Multi-Cloud)

#### Operator-Based Deployment

**Strimzi Operator:**
- Custom resources for Kafka, MM2, topics
- Automatic resource management
- Rolling updates with zero downtime

**Cluster Configuration:**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: site-a-cluster
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
    config:
      broker.rack: "${STRIMZI_BROKER_RACK}"
      min.insync.replicas: 2
      default.replication.factor: 3
    storage:
      type: persistent-claim
      size: 1Ti
      class: fast-ssd
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
```

**MM2 Configuration:**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: site-a-to-site-b
spec:
  version: 3.6.0
  replicas: 3
  connectCluster: "site-b"
  clusters:
    - alias: "site-a"
      bootstrapServers: site-a-kafka-bootstrap:9092
    - alias: "site-b"
      bootstrapServers: site-b-kafka-bootstrap:9092
  mirrors:
    - sourceCluster: "site-a"
      targetCluster: "site-b"
      sourceConnector:
        config:
          replication.factor: 3
          offset-syncs.topic.replication.factor: 3
          sync.group.offsets.enabled: true
      checkpointConnector:
        config:
          checkpoints.topic.replication.factor: 3
      topicsPattern: "orders.*, transactions.*"
      groupsPattern: ".*"
```

**Advantages:**
- Declarative configuration
- Automatic recovery and scaling
- Built-in monitoring integration
- Cross-cloud portability

---

## Testing and Validation Strategies

### Pre-Production Testing

#### Network Latency Simulation

**Purpose:** Validate cluster behavior under various network conditions

**Tools:**
- Linux `tc` (traffic control) for latency injection
- `netem` for network emulation
- Chaos engineering tools (Chaos Mesh, Pumba)

**Test Scenarios:**
```bash
# Add 20ms latency between sites
tc qdisc add dev eth0 root netem delay 20ms

# Add 5% packet loss
tc qdisc add dev eth0 root netem loss 5%

# Add latency variance (jitter)
tc qdisc add dev eth0 root netem delay 20ms 5ms

# Simulate network partition
iptables -A INPUT -s <site-b-network> -j DROP
```

**Validation:**
- Monitor UnderReplicatedPartitions
- Check ISR shrink/expand rates
- Measure write latency increase
- Verify cluster remains available

#### Failover Testing

**Stretch Cluster Failover Test:**

**Test 1: Graceful Single Broker Shutdown**
1. Stop broker in Site A
2. Verify leader election for affected partitions
3. Confirm zero message loss
4. Measure failover time
5. Restart broker and verify ISR rejoin

**Test 2: Network Partition**
1. Block all traffic between sites
2. Observe behavior: cluster should degrade gracefully
3. Verify writes fail if min.insync.replicas not satisfied
4. Restore network
5. Verify automatic recovery

**Test 3: Complete Site Failure**
1. Shutdown all Site A brokers simultaneously
2. Verify Site B continues operating
3. Measure impact on producers/consumers
4. Bring Site A back online
5. Verify full synchronization

**Active-Active Failover Test:**

**Test 1: Planned Failover**
1. Stop producing to Site A
2. Wait for MM2 to catch up (monitor lag)
3. Redirect all traffic to Site B
4. Verify consumer offset translation works
5. Measure end-to-end failover time

**Test 2: Unplanned Failover**
1. Abruptly terminate Site A cluster
2. Applications detect failure and failover
3. Verify message duplication/loss within tolerance
4. Check conflict resolution handles divergence
5. Restore Site A and reconcile

**Test 3: Split-Brain Scenario**
1. Create network partition during active writes
2. Both sites continue accepting writes
3. Restore connectivity
4. Verify conflict resolution resolves divergence
5. Validate no data corruption

#### Performance Testing

**Baseline Performance:**
- Establish baseline metrics for normal operations
- Document p50, p99, p999 latencies
- Record maximum sustainable throughput
- Measure resource utilization

**Degraded Mode Testing:**
- Test with one site degraded (slow disks, CPU throttling)
- Measure impact on overall cluster performance
- Verify SLAs still met in degraded state

**Load Testing:**
- Use tools like kafka-producer-perf-test
- Gradually increase load to find breaking point
- Test with realistic message sizes and patterns
- Monitor system resources throughout

**Example Load Test:**
```bash
# Test producer throughput
kafka-producer-perf-test \
  --topic test-topic \
  --num-records 10000000 \
  --record-size 1024 \
  --throughput 100000 \
  --producer-props \
    bootstrap.servers=broker1:9092 \
    acks=all \
    compression.type=lz4

# Test consumer throughput
kafka-consumer-perf-test \
  --topic test-topic \
  --messages 10000000 \
  --threads 4 \
  --broker-list broker1:9092
```

### Production Validation

#### Health Checks

**Continuous Health Monitoring:**

**Broker Health:**
- JMX metrics collection every 15 seconds
- Alert on UnderReplicatedPartitions > 0
- Alert on controller flapping
- Monitor disk space utilization

**Replication Health:**
- MM2 connector status checks
- Replication lag monitoring
- Checkpoint lag verification
- Network connectivity between sites

**Application Health:**
- Producer success rate
- Consumer lag per group
- Error rate monitoring
- End-to-end latency tracking

#### Disaster Recovery Drills

**Quarterly DR Drill Schedule:**

**Q1: Partial Failure**
- Simulate single broker failure
- Test automatic recovery
- Validate monitoring alerts
- Document lessons learned

**Q2: Site Degradation**
- Throttle network to one site
- Test degraded mode operations
- Measure performance impact
- Validate operational procedures

**Q3: Complete Site Failure**
- Full site failover test
- Test all runbooks
- Involve all teams
- Measure RTO/RPO achieved

**Q4: Split-Brain Scenario**
- Network partition test
- Validate conflict resolution
- Test recovery procedures
- Update documentation

**Drill Validation Criteria:**
- RTO within target (typically <15 minutes)
- RPO within target (typically <60 seconds)
- Zero critical errors during failover
- All systems operational post-recovery
- Documentation complete and accurate

#### Chaos Engineering

**Gradual Chaos Introduction:**

**Level 1: Infrastructure Chaos**
- Random broker restarts
- Network latency injection
- Disk I/O throttling
- CPU throttling

**Level 2: Application Chaos**
- Simulate slow consumers
- Producer failures and retries
- Consumer group rebalances
- Connection drops

**Level 3: Site-Level Chaos**
- Complete site outages
- Network partitions
- Cascading failures
- Recovery scenarios

**Tools:**
- Chaos Mesh (Kubernetes)
- Gremlin (SaaS platform)
- Custom scripts for specific scenarios
- Automated chaos experiments in staging

---

## Operational Best Practices

### Configuration Management

**Version Control:**
- All Kafka configurations in Git
- Broker configurations templated per site
- Topic configurations as code
- MM2 configurations version controlled

**Change Management:**
- Peer review for configuration changes
- Testing in non-production environment
- Gradual rollout (canary changes)
- Rollback procedures documented

**Configuration Consistency:**
- Use configuration management tools (Ansible, Terraform)
- Validate configurations before deployment
- Automated testing of configuration changes
- Regular audits for drift detection

### Capacity Planning

**Broker Capacity:**
- CPU: 70% average utilization maximum
- Memory: 80% utilization maximum
- Disk: 60% utilization maximum for growth
- Network: 70% bandwidth utilization maximum

**Storage Growth:**
```
Daily Growth = Messages/day × Avg Message Size × Retention Days
Safety Factor = 1.3 (30% buffer)
Required Storage = Daily Growth × Safety Factor
```

**Scaling Triggers:**
- CPU sustained >70% for 1 hour
- Disk utilization >60%
- Network saturation events
- Partition count approaching limits

**Partition Calculation:**
```
Partitions = (Target Throughput / Single Partition Throughput) × Safety Factor
Consider: Consumer parallelism, ordering requirements, cost
Typical: 1000-3000 partitions per broker
```

### Security Considerations

**Encryption:**
- TLS for inter-broker communication (essential for stretch cluster)
- TLS for client connections
- Encryption at rest for sensitive data
- Key management strategy (KMS, Vault)

**Authentication:**
- SASL/SCRAM for client authentication
- mTLS for broker-to-broker
- Integration with enterprise identity systems
- Regular credential rotation

**Authorization:**
- ACLs for topic-level access control
- Principle of least privilege
- Regular access reviews
- Audit logging enabled

**Network Security:**
- Firewall rules limiting Kafka ports
- VPN or private links between sites
- DDoS protection for external endpoints
- Regular security assessments

### Upgrade Strategy

**Rolling Upgrade Process:**

**Preparation:**
1. Review release notes for breaking changes
2. Test upgrade in non-production
3. Backup configurations
4. Schedule maintenance window (if needed)
5. Prepare rollback plan

**Execution (Stretch Cluster):**
1. Upgrade brokers one at a time
2. Start with non-controller brokers
3. Wait for ISR stabilization between brokers
4. Monitor metrics during upgrade
5. Upgrade controllers last
6. Verify cluster health

**Execution (Active-Active):**
1. Upgrade one site completely
2. Verify MM2 continues working
3. Monitor replication lag
4. Upgrade second site
5. Verify bidirectional replication

**Validation:**
- Produce and consume test messages
- Check all metrics returned to baseline
- Verify MM2 connectors healthy
- Run integration tests
- Monitor for 24 hours post-upgrade

---

## Troubleshooting Guide

### Common Issues

#### Issue 1: High Replication Lag (Stretch Cluster)

**Symptoms:**
- UnderReplicatedPartitions increasing
- ISR shrinking frequently
- Write latency increasing

**Possible Causes:**
- Network latency increased
- Slow follower broker (disk, CPU)
- Insufficient network bandwidth
- High producer throughput burst

**Investigation:**
```bash
# Check replica lag
kafka-replica-verification.sh --broker-list <brokers>

# Check broker metrics
kafka-broker-api-versions.sh --bootstrap-server broker:9092

# Monitor network latency
ping <remote-site>
mtr <remote-site>

# Check disk I/O
iostat -x 1
```

**Resolution:**
- Increase `replica.lag.time.max.ms` if network consistently slower
- Scale up slow broker (CPU, memory, disk)
- Add network bandwidth between sites
- Throttle producers temporarily
- Review partition distribution

#### Issue 2: MM2 Replication Stopped

**Symptoms:**
- Replication lag increasing continuously
- MM2 connector status FAILED
- Missing topics in target cluster

**Possible Causes:**
- Network connectivity loss
- Authentication failure
- Target cluster capacity exhausted
- Topic creation failure (auto.create.topics.enable=false)

**Investigation:**
```bash
# Check connector status
curl http://mm2-host:8083/connectors/mm2-source/status

# Check connector logs
docker logs mm2-container
kubectl logs mm2-pod

# Test connectivity
kafka-broker-api-versions.sh --bootstrap-server target:9092

# Check target cluster capacity
kafka-broker-api-versions.sh --bootstrap-server target:9092
```

**Resolution:**
- Restart failed connectors
- Verify network connectivity and firewall rules
- Update authentication credentials
- Scale target cluster if capacity issue
- Enable auto.create.topics or manually create topics
- Review connector configuration

#### Issue 3: Consumer Offset Translation Failure

**Symptoms:**
- Consumers reset to beginning after failover
- Duplicate message processing
- Large consumer lag post-failover

**Possible Causes:**
- Checkpoint connector not running
- Checkpoint lag too high
- Consumer group name mismatch
- Checkpoint topic missing

**Investigation:**
```bash
# Check checkpoint connector status
curl http://mm2-host:8083/connectors/mm2-checkpoint/status

# List checkpoint topics
kafka-topics.sh --bootstrap-server target:9092 --list | grep checkpoint

# Check consumer group offsets
kafka-consumer-groups.sh --bootstrap-server source:9092 \
  --group <group-id> --describe
```

**Resolution:**
- Ensure checkpoint connector running
- Reduce checkpoint interval for lower lag
- Verify consumer group names consistent
- Manually translate offsets if checkpoints missing
- Configure consumers for idempotent processing

#### Issue 4: Split-Brain in Stretch Cluster

**Symptoms:**
- Multiple active controllers
- Conflicting partition leaders
- Clients seeing inconsistent data

**Possible Causes:**
- Network partition between sites
- Controller quorum issues
- Configuration mismatch

**Investigation:**
```bash
# Check controller status
kafka-metadata.sh --cluster <cluster-id> --controller

# Check ZooKeeper quorum (if not KRaft)
echo stat | nc zk-host 2181

# Check network connectivity
ping <remote-site>
traceroute <remote-site>
```

**Resolution:**
- Immediate: Shut down minority partition to resolve split-brain
- Verify network connectivity restored
- Check controller quorum configuration
- Restart affected brokers in coordinated manner
- Verify partition leadership converged

#### Issue 5: Disk Full on Broker

**Symptoms:**
- Broker logs errors writing to disk
- Broker marked as offline
- Partitions become unavailable

**Investigation:**
```bash
# Check disk usage
df -h

# Check log directories
du -sh /var/lib/kafka/data/*

# Identify large log segments
find /var/lib/kafka/data -type f -size +1G
```

**Resolution:**
- Immediate: Delete old log segments if retention allows
- Reduce retention time temporarily
- Add disk capacity
- Enable log compaction for appropriate topics
- Implement tiered storage for long retention
- Review partition distribution and rebalance

### Performance Optimization

**Producer Optimization:**
- Increase `batch.size` (default 16KB → 32-64KB)
- Increase `linger.ms` (default 0 → 10-20ms)
- Enable compression (`compression.type=lz4` or `zstd`)
- Tune `buffer.memory` for high throughput
- Use transactions for exactly-once semantics

**Consumer Optimization:**
- Increase `fetch.min.bytes` to batch fetches
- Tune `max.poll.records` for processing capacity
- Use multiple consumers in same group for parallelism
- Optimize `session.timeout.ms` and `heartbeat.interval.ms`
- Consider consumer-side caching for repeated reads

**Broker Optimization:**
- Use `num.network.threads=8` and `num.io.threads=16` as starting point
- Tune `socket.send.buffer.bytes` and `socket.receive.buffer.bytes`
- Enable `log.cleaner` for compacted topics
- Use `compression.type` at broker level if needed
- Monitor and tune JVM heap (typically 6-8GB)

**Network Optimization:**
- Enable jumbo frames (MTU 9000) if supported
- Use dedicated network interfaces for replication
- Implement QoS policies prioritizing Kafka traffic
- Monitor for network saturation and upgrade capacity
- Use network bonding/teaming for redundancy

---

## Summary and Decision Matrix

### Architecture Selection Guide

**Choose Stretch Cluster if:**
- Data centers in same metro area (<100mi)
- Low-latency network available (<10ms RTT)
- Strong consistency required
- Simplest operations preferred
- Acceptable write latency increase (10-30ms)

**Choose Active-Active if:**
- Sites geographically distributed (>100mi)
- Local low-latency operations required
- Application supports conflict resolution
- High availability critical
- Network partitions expected

**Choose Active-Passive if:**
- Simple DR solution needed
- Cost optimization important
- Application cannot handle conflicts
- Backup resources acceptable underutilized
- Manual failover acceptable

**Choose Aggregate Cluster if:**
- Global application with regional operations
- Heavy analytics requirements
- Data sovereignty compliance needed
- Centralized reporting critical
- Budget supports three clusters

**Choose Hub-and-Spoke if:**
- Many edge locations (>5)
- Centralized data aggregation needed
- Edge locations operate independently
- Inter-edge communication not critical
- Clear hub-centric model

**Choose Observer Cluster if:**
- Analytics impact production concern
- Development/testing isolation needed
- Compliance requires separate audit trail
- Long retention on different hardware
- Read-only workloads substantial

### Key Takeaways

1. **No Universal Solution**: Architecture depends on requirements, constraints, budget
2. **Start Simple**: Begin with simpler architecture, evolve as needed
3. **Test Thoroughly**: Invest heavily in testing and validation before production
4. **Monitor Extensively**: Comprehensive monitoring essential for all architectures
5. **Plan for Failure**: Design and test failure scenarios regularly
6. **Document Everything**: Runbooks, configurations, decisions must be documented
7. **Automate Operations**: Manual processes don't scale or work during incidents
8. **Security First**: Encryption, authentication, authorization not afterthoughts
9. **Capacity Planning**: Plan for growth, monitor trends, scale proactively
10. **Team Readiness**: Ensure team trained on architecture and procedures
