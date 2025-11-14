# Kafka on Strimzi across two metro sites: Architecture, Roadmap, and Test Plan

This document outlines deployment options, a phased roadmap, performance and disaster testing strategies, and a reproducible local lab to validate a stretched Kafka deployment with Strimzi on a Kubernetes cluster spanning two geographically separated sites in the same metro region.

---

## 1) Context and assumptions

- Two physical sites in one metro region, low-latency link (typical RTT 0.5–5 ms), non-zero risk of network partitions.
- A single “stretched” Kubernetes cluster with nodes in both sites (shared control plane).
- Storage is site-local (NVMe or fast block). Cross-site synchronous storage replication is not assumed (Kafka prefers local disks).
- Latest supported Strimzi and Kafka versions (KRaft supported); ZooKeeper still supported if legacy constraints require it.
- DNS can route clients to site-local brokers; TLS and SASL used for transport security and auth.

Non-goals:
- No reliance on proprietary features (e.g., Confluent Cluster Linking).
- No assumption of a third physical site unless explicitly stated.

---

## 2) Architecture options in a 2-site metro stretch

You cannot have a single Kafka cluster that both a) keeps write availability and b) guarantees RPO≈0 during a full site failure using only two sites and standard Kafka quorum rules. You must trade off.

Option A: Single stretched Kafka cluster (KRaft) with controllers pinned to one site
- Summary: One Kafka cluster across both sites; all KRaft controllers in Site A; brokers spread across both sites via rack-awareness.
- Pros: Simpler ops; minimal client changes; can survive node/rack failures; can run with acks=all, min.insync.replicas=2.
- Cons: If Site A (controllers) fails, cluster becomes unavailable (RTO depends on rebuild/re-point of controllers).
- RPO during site loss: N/A (cluster down if controller site is lost).
- Write availability during full site failure: No (if controller site fails).

Option B: Single stretched Kafka cluster with controller quorum across 3 zones (requires a “witness”)
- Summary: 3 or 5 KRaft controllers spread across Site A, Site B, and a tiny witness zone (third failure domain).
- Pros: Can keep metadata quorum through loss of one site; preserves cluster availability with proper placement.
- Cons: Requires third failure domain; cross-zone latency impacts controller commit latency; increased operational complexity.
- Write availability during full site failure: Yes (with correct ISR placement and surviving majority).
- RPO during site loss: 0 for committed data with acks=all and sufficient ISR in surviving site.

Option C: Two independent Kafka clusters (one per site) + MirrorMaker 2 (active-active or active-passive)
- Summary: Two clusters; replicate topics and consumer offsets with MM2; fail clients over between clusters.
- Pros: Survives total loss of either site; maintenance isolation; decoupled quorum issues.
- Cons: Client failover logic required; duplicate topics/state across clusters; exactly-once across sites not trivial.
- RPO: Configurable; with acks=all and MM2 lag under control, near-zero for most topics; still asynchronous.
- Write availability during full site failure: Yes (route to surviving site).

Decision guidance:
- If you must keep service through a full site failure and don’t have a third failure domain: choose Option C (dual clusters with MM2).
- If you can tolerate an outage on total loss of one site but want simplicity: choose Option A (single stretched cluster, controllers in one site).
- If you have a third failure domain: Option B gives single-cluster semantics with site-level resilience.

---

## 3) Kubernetes and Strimzi layout

Node and topology labeling:
- Label nodes by site:
  - topology.kubernetes.io/region: metro-1
  - topology.kubernetes.io/zone: site-a or site-b (and witness if present)
- StorageClasses per site with WaitForFirstConsumer volume binding.
- Topology spread constraints for Pods to balance across nodes within a site.

Strimzi knobs relevant to stretch:
- Rack awareness: spec.kafka.rack.topologyKey: "topology.kubernetes.io/zone"
- Listeners: at least one internal listener; optional external per site for direct client access.
- Replication and ISR:
  - default.replication.factor: 3 (or 4 if you need more ISR headroom)
  - min.insync.replicas: 2 (lower latency trade-off; set to 3 for tighter RPO but more write sensitivity)
  - unclean.leader.election.enable: false
  - offsets.topic.replication.factor: 3
  - transaction.state.log.replication.factor: 3
  - transaction.state.log.min.isr: 2

Cruise Control and KafkaRebalance:
- Enable Cruise Control for automated rebalancing, rack-aware balancing, and to test rebalance under failures.

PodDisruptionBudgets:
- Ensure PDBs are set so voluntary disruptions never evict too many brokers at once.

Network policies:
- Constrain cross-namespace access; later reuse to simulate partitions in tests.

---

## 4) Example manifests (trimmed, adjust to your Strimzi version)

4.1 Single stretched cluster (ZooKeeper-based; widely compatible)

Note: Prefer KRaft if your org is ready; zk manifests are included because they’re stable and familiar.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: metro
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 6
    rack:
      topologyKey: topology.kubernetes.io/zone
    listeners:
      - name: internal
        port: 9092
        type: internal
        tls: true
    config:
      num.partitions: 12
      default.replication.factor: 3
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      unclean.leader.election.enable: false
    storage:
      type: persistent-claim
      size: 500Gi
      class: fast-ssd
      deleteClaim: false
  zookeeper:
    replicas: 5  # Spread across Site A, Site B, and witness if available; else accept trade-off
    storage:
      type: persistent-claim
      size: 50Gi
      class: fast-ssd
  entityOperator:
    topicOperator: {}
    userOperator: {}
  cruiseControl:
    config:
      goals: "RackAwareGoal,ReplicaCapacityGoal,DiskCapacityGoal,NetworkInboundCapacityGoal,NetworkOutboundCapacityGoal,ReplicaDistributionGoal,LeaderReplicaDistributionGoal,TopicReplicaDistributionGoal"
```

4.2 Single stretched cluster (KRaft with NodePools; indicative)

Note: Exact fields depend on your Strimzi release. Use NodePools to separate controllers and brokers and to label per-site broker pools.

```yaml
# Kafka cluster (no ZooKeeper)
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: metro
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    rack:
      topologyKey: topology.kubernetes.io/zone
    listeners:
      - name: internal
        port: 9092
        type: internal
        tls: true
    config:
      num.partitions: 12
      default.replication.factor: 3
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      unclean.leader.election.enable: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
  cruiseControl: {}
---
# Controllers pool (spread across 3 zones if you have a witness)
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: metro-controllers
  namespace: kafka
  labels:
    strimzi.io/cluster: metro
spec:
  replicas: 5
  roles:
    - controller
  storage:
    type: persistent-claim
    size: 30Gi
    class: fast-ssd
  template:
    pod:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["site-a", "site-b", "witness"]
---
# Brokers in Site A
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: metro-brokers-a
  namespace: kafka
  labels:
    strimzi.io/cluster: metro
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: persistent-claim
    size: 500Gi
    class: fast-ssd
  template:
    pod:
      metadata:
        labels:
          site: a
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["site-a"]
---
# Brokers in Site B
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: metro-brokers-b
  namespace: kafka
  labels:
    strimzi.io/cluster: metro
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: persistent-claim
    size: 500Gi
    class: fast-ssd
  template:
    pod:
      metadata:
        labels:
          site: b
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["site-b"]
```

4.3 Dual-cluster with MirrorMaker 2

```yaml
# Site A cluster (name: a)
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: a
  namespace: kafka-a
spec:
  kafka:
    version: 3.7.0
    replicas: 3
    rack:
      topologyKey: topology.kubernetes.io/zone
    listeners:
    - name: internal
      port: 9092
      type: internal
      tls: true
    config:
      default.replication.factor: 3
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      unclean.leader.election.enable: false
    storage:
      type: persistent-claim
      size: 500Gi
      class: fast-ssd
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 50Gi
      class: fast-ssd
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
# Site B cluster (name: b) similar spec in namespace kafka-b
```

MirrorMaker 2 (in either site or a neutral namespace):
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: mm2
  namespace: mm2
spec:
  replicas: 2
  connectCluster: "b"
  clusters:
    - alias: "a"
      bootstrapServers: "a-kafka-bootstrap.kafka-a.svc:9092"
      tls:
        trustedCertificates:
          - secretName: a-cluster-ca-cert
            certificate: ca.crt
    - alias: "b"
      bootstrapServers: "b-kafka-bootstrap.kafka-b.svc:9092"
      tls:
        trustedCertificates:
          - secretName: b-cluster-ca-cert
            certificate: ca.crt
  mirrors:
    - sourceCluster: "a"
      targetCluster: "b"
      topicsPattern: ".*"
      groupsPattern: ".*"
      sourceConnector: {}
      checkpointConnector: {}
      heartbeatConnector: {}
      config:
        replication.factor: 3
        sync.group.offsets.enabled: true
        emit.checkpoints.interval.seconds: 5
        refresh.topics.enabled: true
        tasks.max: 8
```

---

## 5) Performance test strategy

Workload profiles:
- Small messages (0.5–1 KB), medium (10–100 KB), large (1–5 MB).
- Producer acks: 1 vs all; linger.ms and batch.size tuning.
- Compression: none vs lz4/zstd.
- Consumer patterns: read-your-writes within site; cross-site reads.

Key metrics to capture:
- Producer ack latency p50/p95/p99
- End-to-end produce→consume latency p50/p95/p99
- Throughput (MB/s and records/s) per topic and cluster
- ISR size distribution, under-replicated partitions
- Controller and broker CPU/heap, GC pause
- Disk I/O: read/write bandwidth and latency per broker
- Network: cross-site bandwidth utilization and retransmits
- Consumer lag and rebalance frequency
- Cruise Control recommended proposals

Tools:
- Kafka perf tools: kafka-producer-perf-test.sh, kafka-consumer-perf-test.sh
- kcat (kafkacat) for quick probes
- OpenMessaging Benchmark (OMB) for sustained load (optional)
- Prometheus + Grafana dashboards (Strimzi provides metrics endpoints)
- Cruise Control UI or API

Scenarios:
- Baseline (no faults), local site producers/consumers only
- Cross-site produce/read
- TLS on/off (if allowed in lab) to quantify crypto overhead
- Scale partitions and producers to max target throughput
- Rebalance during load (KafkaRebalance CR)
- Rolling upgrade during load (Strimzi operator rolling)

Acceptance examples (adjust to SLOs):
- p99 producer acks=all latency ≤ 15 ms under normal load
- No under-replicated partitions at steady state
- Consumer lag recovers to baseline within 60 s after a single-broker loss
- Rebalance completes < 15 min for N partitions at target throughput

---

## 6) Disaster and chaos test matrix

Failure modes to test:
1) Single broker crash
- Method: kubectl delete pod
- Expectation: No data loss; ISR shrinks then recovers; client retries succeed; CC proposes rebalances optionally.

2) Single node loss
- Method: stop node or taint+delete pods
- Expectation: Similar to broker crash; slower if multiple brokers on node.

3) Controller loss (KRaft) or ZooKeeper node loss
- Method: delete controller Pod(s)
- Expectation: Metadata quorum maintained; no write outage.

4) Network delay increase between sites
- Method: inject 2–5 ms RTT via Chaos Mesh NetworkChaos
- Expectation: Mild increase in replication latency; acks=all p99 increases but steady.

5) Network partition between sites
- Method: NetworkPolicy or Chaos Mesh partition
- Expectation (single-cluster): acks=all may fail if ISR crosses sites and min.insync.replicas not met; reads ok from surviving replicas; no unclean leader election.
- Expectation (dual-cluster): each site cluster unaffected; MM2 replication pauses.

6) Disk pressure / slow disk on a subset of brokers
- Method: stress/fio inside broker Pod namespace
- Expectation: Replication lag increases; CC may rebalance leaders away.

7) Entire site outage
- Method: cordon+drain all nodes in site, or stop node VM group
- Expectation:
  - Option A: cluster unavailable if controller site lost; if not, writes may continue if min.insync.replicas satisfied.
  - Option C: fail clients to surviving site; MM2 later reverses direction for failback.

8) Rolling upgrade of Kafka and Strimzi during load
- Expectation: No data loss; bounded latency spikes; no stuck rolling.

Pass/fail criteria:
- No unclean leader election
- No data loss with acks=all
- Meets defined RTO/RPO per chosen topology
- Alerting triggers as expected (noisy alerts triaged)

---

## 7) Reproducible local lab (single laptop)

Goal: Safely simulate stretch behavior without impacting production.

Approach A: One multi-node cluster (kind) with sites as node labels
- Good for single-cluster (Options A/B) validation.

Approach B: Two clusters (k3d or kind) to validate MM2 (Option C)
- Good for full site outage and failover drills.

Prerequisites:
- Docker, kubectl, Helm, kind (or k3d), GNU make, jq
- OS: Linux/macOS (Linux preferred for tc/netem)

7.1 Create a 2-site kind cluster

kind config:
```yaml
# kind-two-sites.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: metro
nodes:
- role: control-plane
- role: worker
  labels:
    topology.kubernetes.io/region: metro-1
    topology.kubernetes.io/zone: site-a
- role: worker
  labels:
    topology.kubernetes.io/region: metro-1
    topology.kubernetes.io/zone: site-a
- role: worker
  labels:
    topology.kubernetes.io/region: metro-1
    topology.kubernetes.io/zone: site-b
- role: worker
  labels:
    topology.kubernetes.io/region: metro-1
    topology.kubernetes.io/zone: site-b
```

Commands:
```bash
kind create cluster --config kind-two-sites.yaml
kubectl create ns kafka
```

Install Strimzi via Helm:
```bash
helm repo add strimzi https://strimzi.io/charts/
helm repo update
helm upgrade --install strimzi strimzi/strimzi-kafka-operator -n kafka --set watchNamespaces="{kafka}"
```

Deploy the Kafka CR (pick KRaft or ZK example above; start with ZK for simplicity in lab):
```bash
kubectl -n kafka apply -f kafka-metro.yaml
kubectl -n kafka rollout status statefulset/metro-kafka
```

7.2 Install observability
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prom prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
# Enable Strimzi metrics as per docs (ServiceMonitors). Use prebuilt Grafana dashboards for Kafka/Strimzi.
```

7.3 Inject cross-site latency and partitions (Chaos Mesh)
```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

Network delay (approximate) from Site B brokers to Site A brokers (requires per-site pod labels; see NodePools example that sets pod label site: a/b):
```yaml
# cross-site-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: cross-site-delay
  namespace: chaos-mesh
spec:
  action: delay
  mode: all
  selector:
    labelSelectors:
      "site": "b"
    namespaces: ["kafka"]
  delay:
    latency: "3ms"
    correlation: "0"
    jitter: "1ms"
  direction: to
  target:
    selector:
      labelSelectors:
        "site": "a"
      namespaces: ["kafka"]
    mode: all
```

Apply:
```bash
kubectl apply -f cross-site-delay.yaml
```

Network partition between sites:
```yaml
# cross-site-partition.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: cross-site-partition
  namespace: chaos-mesh
spec:
  action: partition
  mode: all
  selector:
    labelSelectors:
      "site": "a"
    namespaces: ["kafka"]
  direction: to
  target:
    selector:
      labelSelectors:
        "site": "b"
      namespaces: ["kafka"]
    mode: all
```

7.4 Load generation
Run a client Pod:
```bash
kubectl -n kafka run ktools --image=ghcr.io/edenhill/kcat:1.7.1 -it --restart=Never -- bash
```

Create a topic and run perf test (in another pod based on Kafka image or in a small job):
```bash
# Producer perf (from a Kafka tools pod)
kafka-topics.sh --bootstrap-server metro-kafka-bootstrap:9092 --command-config /tmp/client.properties --create --topic perf1 --partitions 24 --replication-factor 3

kafka-producer-perf-test.sh --throughput -1 --num-records 2000000 --record-size 1000 \
  --producer.config /tmp/client.properties --topic perf1 --print-metrics \
  --producer-props acks=all linger.ms=5 batch.size=131072 compression.type=lz4
```

7.5 Simulate failures

- Broker crash:
```bash
kubectl -n kafka delete pod -l strimzi.io/name=metro-kafka --field-selector=status.phase=Running --wait=false --grace-period=0 --limit=1
```

- Site outage (approximate in kind): evict all site-a brokers and block cross-site traffic
```bash
# Label selector relies on pod label site=a
kubectl -n kafka delete pod -l site=a --force --grace-period=0
kubectl -n chaos-mesh apply -f cross-site-partition.yaml
```

- Disk pressure on one broker (slow I/O simulation)
```bash
kubectl -n kafka exec -it metro-kafka-0 -- bash -lc "apt-get update && apt-get install -y stress-ng && stress-ng --hdd 1 --hdd-opts sync --timeout 120s"
```

7.6 Measure and record
- Use Grafana dashboards: Kafka/Brokers, Kafka Exporter, JVM, Kubernetes Node.
- Capture p99 producer latency deltas under each fault.
- Export Cruise Control proposals pre/post fault.

7.7 Tear down chaos
```bash
kubectl -n chaos-mesh delete networkchaos cross-site-delay cross-site-partition
```

Approach B (two clusters with MM2)
- Create two kind or k3d clusters: metro-a and metro-b.
- Install Strimzi in both; deploy Kafka in each; deploy MM2 in either cluster or a third namespace.
- Produce to cluster A; verify replication to B; stop cluster A (k3d cluster stop metro-a); verify client failover by changing bootstrap to B; measure RPO (messages not yet replicated).

---

## 8) Rollout roadmap

Phase 0: Goals and SLOs
- Define RPO/RTO, throughput/latency SLOs, and failover requirements (write availability under site loss or not).

Phase 1: Lab validation (local)
- Stand up kind/k3d lab; run performance baselines; execute full chaos matrix; produce a runbook.

Phase 2: Staging (cloud/lab hardware)
- Deploy the chosen topology (Option A/B/C); set up observability; validate infra limits; dry-run rolling upgrades.

Phase 3: Client integration
- Implement multi-bootstrap client configs (site-local first, remote as fallback).
- Configure producer retries, idempotence, and timeouts per SLO.

Phase 4: Production canary
- Canary topics and clients; enable Cruise Control; soak tests; validate alert thresholds.

Phase 5: Production
- Full rollout; controlled traffic ramp-up; schedule regular DR drills (quarterly) and capture metrics.

---

## 9) Operational notes and tunables

- Producer
  - acks=all, enable.idempotence=true for lossless guarantee
  - delivery.timeout.ms aligned with site-failure detection policy
  - retries and backoff tuned to expected ISR churn
- Broker
  - replica.lag.time.max.ms aligned with cross-site RTT
  - leader.throttled.replicas and follower.throttled.replicas for controlled rebalances
- Controller quorum (KRaft)
  - Prefer 3 or 5 controllers; place for majority survivability
- Storage
  - xfs, noatime; fs tuned for large journals; consider page cache sizing
- Security
  - mTLS within cluster; rotate certs with Strimzi CAs; SASL OAUTHBEARER or SCRAM if needed
- Observability
  - Alert on UnderReplicatedPartitions > 0 (latched), OfflinePartitionsCount, ControllerUnavailable, high P95 produce latency, GC pause > threshold, disk usage > 80%

---

## 10) Failover runbooks (high level)

Single stretched cluster (Option A/B):
- Loss of a broker: no client action; monitor ISR; rebalance after recovery.
- Partition between sites: if producers hit NotEnoughReplicas, back off/retry; optionally lower min.insync.replicas for specific topics during incident (only if acceptable).
- Loss of controller site (Option A): declare outage; rebuild/reassign controllers or promote DR cluster if present.

Dual cluster with MM2 (Option C):
- Loss of Site A:
  - Switch client bootstrap to Site B; ensure topics exist (pre-provisioned) and ACLs in place.
  - Promote B as write primary; checkpoint consumer offsets with MM2 if available; realign consumer groups.
- Failback:
  - Re-enable MM2 from B→A; wait for catch-up; switch writers back; reverse replication as needed.

---

## 11) Risks and mitigations

- Two-site quorum limitations (no third witness): cannot guarantee both write availability and RPO≈0 under full site loss with a single cluster. Mitigate via dual clusters (MM2) or add a third failure domain for controllers.
- Cross-site latency variability: increases p99 lat; mitigate via batching, linger, compression, and careful ISR/min.insync settings.
- Operator and upgrade risks: rehearse upgrades in staging; pin versions; use maintenance windows.
- Human error during DR: automate with scripts/runbooks; test quarterly.

---

## 12) Summary decision matrix

| Topology | Survives full site outage | Single-cluster simplicity | Client failover complexity | Recommended when |
|---|---|---|---|---|
| Single cluster, controllers in one site (A) | No (if A lost) | High | Low | Simpler ops; outage acceptable on full site loss |
| Single cluster with witness (3 zones) | Yes | Medium | Low | Third failure domain exists; want single-cluster semantics |
| Two clusters + MM2 | Yes | Lower | Medium/High | No witness site; need write availability through site loss |

---

If you want, I can provide a ready-to-run repo skeleton (manifests, Makefile, chaos scenarios, Grafana dashboards) tailored to your exact Strimzi/Kafka versions and SLOs.