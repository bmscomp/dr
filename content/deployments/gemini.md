
## Part 1: Roadmap and Automation of Deployment

This roadmap outlines the phased deployment of a Kafka on Strimzi cluster across two sites, including the critical architectural decisions and automated deployment manifests.

### 1.1 Critical Architecture Decision

Before deployment, you must choose between two primary architectures. This decision depends entirely on your requirements for write availability versus operational simplicity during a full site failure.

| Topology | Survives Full Site Outage | Single-Cluster Simplicity | Client Failover Complexity | Recommended When... |
| :--- | :--- | :--- | :--- | :--- |
| **Single Stretched Cluster** (Option A/B) | **No** (if controller site lost) or **Yes** (if 3-zone witness exists) | High | Low | Simpler ops are prioritized, and a brief outage on full site loss is acceptable OR a third witness site is available. |
| **Two Clusters + MirrorMaker 2** (Option C) | **Yes** | Lower | Medium/High | No witness site exists, and **write availability must be maintained** through a full site loss. |

This guide provides automation steps for both options.

-----

### 1.2 Phased Deployment Roadmap

This roadmap follows a phased approach to de-risk the rollout.

  * **Phase 1: Planning & Design**

      * Define RPO/RTO targets and latency/throughput SLOs.
      * Finalize the architecture (Single Stretched vs. Dual Cluster).
      * Define network requirements (VLANs, QoS) and storage classes per site.
      * Calculate storage and resource requirements.

  * **Phase 2: Lab & Infrastructure Preparation**

      * Build a local lab (see Annex 1) to validate the chosen topology and run chaos tests.
      * Prepare the production Kubernetes cluster nodes with site-specific labels for rack awareness. Use `topology.kubernetes.io/zone` for the site/rack identifier.
      * Prepare site-specific StorageClasses with `WaitForFirstConsumer` binding.

  * **Phase 3: Strimzi Deployment & Automation**

      * Automate the deployment of the Strimzi Operator and the Kafka cluster itself using the manifests in the next section.
      * Deploy monitoring (Prometheus/Grafana) and observability tools.

  * **Phase 4: Validation & Client Integration**

      * Run performance baseline tests (see Annex 2) in the staging/production environment.
      * Execute the DR test matrix (see Annex 2) on staging hardware.
      * Implement multi-bootstrap client configurations to be site-aware.

  * **Phase 5: Production Canary & Rollout**

      * Migrate non-critical canary topics and clients to the new cluster.
      * Enable Cruise Control for automated rebalancing.
      * Begin progressive rollout to all workloads.
      * Schedule regular DR drills.

-----

### 1.3 How to Automate Deployment

Automation is achieved by defining your infrastructure as code using Kubernetes manifests.

#### 1.3.1 Kubernetes Node Labeling (Prerequisite)

All nodes in your stretched cluster must be labeled by site. This is the key to enabling rack awareness.

```yaml
# Example labels for a worker node in Site A
labels:
  topology.kubernetes.io/region: metro-1
  topology.kubernetes.io/zone: site-a 
  kafka.strimzi.io/rack: site-a # Optional, but good practice [3]

# Example labels for a worker node in Site B
labels:
  topology.kubernetes.io/region: metro-1
  topology.kubernetes.io/zone: site-b
  kafka.strimzi.io/rack: site-b # Optional, but good practice [3]
```

#### 1.3.2 Strimzi Operator Installation (Helm)

```bash
# Add the Strimzi Helm repo
helm repo add strimzi https://strimzi.io/charts/
helm repo update

# Install the operator in its own namespace
helm upgrade --install strimzi strimzi/strimzi-kafka-operator \
  -n kafka-operator --create-namespace \
  --set watchNamespaces="{kafka}" # Watch the namespace you will deploy Kafka into
```

-----

#### 1.3.3 Option A: Automated Single Stretched Cluster (KRaft)

This architecture uses **KafkaNodePools** to explicitly pin controllers and brokers to different sites or zones, providing fine-grained control.

**1. The `Kafka` Custom Resource (CR)**
This defines the shared cluster configuration. Note `rack.topologyKey` is set to `topology.kubernetes.io/zone`.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: metro-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    # This 'rack' config is the key to site-awareness
    rack:
      topologyKey: "topology.kubernetes.io/zone"
    listeners:
      - name: internal
        port: 9092
        type: internal
        tls: true
    config:
      default.replication.factor: 3
      min.insync.replicas: 2 # Critical tuning parameter [1, 3]
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      unclean.leader.election.enable: false
    storage:
      type: jbod # 'jbod' is used when using NodePools
      volumes:
      - id: 0
        type: ephemeral # Storage is defined in the NodePools
        size: 1Gi 
  entityOperator:
    topicOperator: {}
    userOperator: {}
  cruiseControl: {}
```

**2. The `KafkaNodePool` for Controllers (3-Zone Witness)**
This example assumes a **3-zone** setup (Site A, Site B, and a Witness) for the controller quorum. This is the *only* way a single cluster can survive a full site failure.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: metro-controllers
  namespace: kafka
  labels:
    strimzi.io/cluster: metro-cluster # Must match the Kafka CR name
spec:
  replicas: 5 # e.g., 2 in site-a, 2 in site-b, 1 in witness
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
```

**3. The `KafkaNodePool` for Brokers (Per-Site)**
These manifests use `nodeAffinity` to ensure brokers are automatically deployed to nodes with the correct site label.

```yaml
# Brokers in Site A
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: metro-brokers-a
  namespace: kafka
  labels:
    strimzi.io/cluster: metro-cluster
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: persistent-claim
    size: 500Gi
    class: site-a-storage # Site-specific storage class
  template:
    pod:
      metadata:
        labels:
          site: a # Pod label for chaos testing
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
    strimzi.io/cluster: metro-cluster
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: persistent-claim
    size: 500Gi
    class: site-b-storage # Site-specific storage class
  template:
    pod:
      metadata:
        labels:
          site: b # Pod label for chaos testing
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["site-b"]
```

-----

#### 1.3.4 Option B: Automated Dual Cluster + MirrorMaker 2

This architecture involves two independent clusters. Automation is achieved by deploying two separate `Kafka` CRs and one `KafkaMirrorMaker2` CR.

**1. Kafka Cluster in Site A**
(Deployed in `kafka-a` namespace, nodes pinned to `site-a`)

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: cluster-a
  namespace: kafka-a
spec:
  kafka:
    version: 3.7.0
    replicas: 3
    # Rack awareness within the site
    rack:
      topologyKey: "kubernetes.io/hostname"
    listeners:
    - name: internal
      port: 9092
      type: internal
      tls: true
    config:
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: persistent-claim
      size: 500Gi
      class: site-a-storage
  # ... ZK, Entity Operator, etc. ...
```

**2. Kafka Cluster in Site B**
(Deployed in `kafka-b` namespace, nodes pinned to `site-b`)

  * *(Manifest is identical to Cluster A, but with `name: cluster-b`, `namespace: kafka-b`, and `class: site-b-storage`)*.

**3. MirrorMaker 2 Deployment**
This CR automates the replication from Site A to Site B. It is typically deployed in a neutral namespace or in the *target* (DR) cluster.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: mm2-a-to-b
  namespace: mm2
spec:
  replicas: 2
  connectCluster: "cluster-b" # Target cluster
  clusters:
    # Source Cluster
    - alias: "cluster-a"
      bootstrapServers: "cluster-a-kafka-bootstrap.kafka-a.svc:9092"
      # ... TLS truststore config ... [1]
    # Target Cluster
    - alias: "cluster-b"
      bootstrapServers: "cluster-b-kafka-bootstrap.kafka-b.svc:9092"
      # ... TLS truststore config ... [1]
  mirrors:
    - sourceCluster: "cluster-a"
      targetCluster: "cluster-b"
      topicsPattern: ".*" # Replicate all topics
      groupsPattern: ".*" # Replicate consumer groups
      config:
        replication.factor: 3
        sync.group.offsets.enabled: true
        # Override renaming policy to keep topic names the same [2]
        replication.policy.separator: ""
        source.cluster.alias: ""
```

-----

## Annex 1: Local Disaster Recovery Tests (Single Machine)

This plan simulates the 2-site environment on a single local machine using `kind` (Kubernetes in Docker) to validate the deployment and automation scripts.

### 2.1 Local Lab Setup

**Step 1: Create a Multi-Node `kind` Cluster**
This `kind` config creates a cluster with 4 worker nodes, labeling 2 as `site-a` and 2 as `site-b` to simulate the two-site topology.

```yaml
# kind-two-sites.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: metro-stretch
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

```bash
# Create the local cluster
kind create cluster --config kind-two-sites.yaml
```

**Step 2: Install Strimzi and Kafka**
Follow the installation steps from Part 1.3.2. Then, apply your chosen Kafka manifest (e.g., Option A: Single Stretched Cluster) from Part 1.3.3. Strimzi will automatically place the pods on the correct `kind` nodes based on their `site-a` / `site-b` labels.

**Step 3: Install Chaos Mesh**
This tool is used to inject failures.

```bash
# Add the Chaos Mesh Helm repo
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update

# Install Chaos Mesh
helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
  -n chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

### 2.2 Automated Local Test Scenarios

These scripts and manifests simulate common disaster scenarios.

#### Test 1: Single Broker Failure

This test validates that the cluster remains available and data is safe when one broker pod is forcibly deleted.

```bash
#!/bin/bash
# simulate-broker-failure.sh
NAMESPACE="kafka"
CLUSTER_NAME="metro-cluster"

echo "=== Starting Single Broker Failure Simulation ==="
# Get a random broker pod from site-a
BROKER_POD=$(kubectl get pods -n $NAMESPACE \
  -l strimzi.io/cluster=$CLUSTER_NAME,strimzi.io/kind=Kafka,site=a \
  -o jsonpath='{.items[0].metadata.name}')

echo "Target broker: $BROKER_POD"
echo "Injecting failure in 5 seconds..."
sleep 5

# Delete pod
kubectl delete pod -n $NAMESPACE $BROKER_POD --force --grace-period=0
echo "Failure injected. Monitoring recovery..."

# Wait for the pod to be recreated and become Ready
kubectl wait --for=condition=Ready pod/$BROKER_POD -n $NAMESPACE --timeout=300s
echo "Pod recovered. Checking cluster health."
echo "=== Simulation Complete ==="
```

#### Test 2: Full Site Failure

This test uses a Chaos Mesh manifest to simulate a complete failure of all pods in `site-a` for 5 minutes.

**Chaos Manifest (`chaos-site-failure.yaml`):**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: site-a-failure
  namespace: chaos-mesh
spec:
  action: pod-failure
  mode: all
  duration: "5m"
  selector:
    namespaces:
      - kafka
    labelSelectors:
      "site": "a" # Relies on the pod label 'site: a' from Part 1.3.3
```

**Test Script:**

```bash
#!/bin/bash
# simulate-site-failure.sh
echo "=== Starting Site-A Failure Simulation ==="

# Start a producer in the background to monitor impact
kubectl run kafka-producer -n kafka --image=quay.io/strimzi/kafka:latest-kafka-3.7.0 \
  --restart=Never -- /bin/sh -c "kafka-producer-perf-test.sh \
  --topic dr-test-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput 100 \
  --producer-props bootstrap.servers=metro-cluster-kafka-bootstrap:9092 acks=all" &

PRODUCER_PID=$!
sleep 10

echo "Injecting site failure..."
kubectl apply -f chaos-site-failure.yaml

echo "Monitoring cluster during failure (5 mins)..."
kubectl get pods -n kafka -l strimzi.io/cluster=metro-cluster -o wide --watch &
WATCH_PID=$!
sleep 300 # Wait for the chaos duration
kill $WATCH_PID

echo "Removing chaos..."
kubectl delete -f chaos-site-failure.yaml

echo "Monitoring recovery..."
kubectl wait --for=condition=Ready pods -n kafka -l site=a --timeout=600s

kill $PRODUCER_PID 2>/dev/null
echo "=== Simulation Complete ==="
```

#### Test 3: Network Partition Between Sites

This test simulates a split-brain scenario where `site-a` cannot communicate with `site-b`. This is the most critical test for a stretched cluster.

**Chaos Manifest (`chaos-network-partition.yaml`):**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-partition
  namespace: chaos-mesh
spec:
  action: partition
  mode: all
  duration: "3m"
  selector:
    namespaces: ["kafka"]
    labelSelectors:
      "site": "a"
  direction: both
  target:
    mode: all
    selector:
      namespaces: ["kafka"]
      labelSelectors:
        "site": "b"
```

-----

## Annex 2: Local + Distant Machine DR Tests

This test plan assumes your cluster is deployed across two *physically separate* sites (e.g., one local, one distant). The goal is to test the real-world impact of network latency and partition, and to practice a full DR failover.

### 3.1 Test 1: Network Latency Impact

**Objective:**
Measure the impact of real-world network latency on producer `acks=all` latency and cluster throughput.

**Procedure:**

1.  Run a baseline performance test (e.g., `kafka-producer-perf-test.sh`) with `acks=all` from a client in `site-a` to the cluster.
2.  Capture the p50, p95, and p99 producer latencies.
3.  **Expected Result:** The p99 latency will be higher than a single-site cluster and should be roughly aligned with your cross-site network RTT.
4.  **Acceptance Criteria:** `p99 producer acks=all latency ≤ 15 ms` (or your defined SLO) under normal load.

### 3.2 Test 2: Full Network Partition

**Objective:**
Validate cluster behavior when the network link between the local and distant site is completely severed.

**Procedure:**

1.  Apply a firewall rule (or equivalent) to block all traffic between the `site-a` and `site-b` Kubernetes nodes.
2.  Run producers and consumers against the cluster.
3.  **Monitor Expected Behavior (Single Stretched Cluster):**
      * If you have a 3-zone controller quorum (A, B, Witness), the majority sites will maintain quorum.
      * One site (the minority) will lose leadership.
      * Producers with `acks=all` may fail with `NotEnoughReplicas` if their partition's In-Sync Replicas (ISR) were spread across both sites and `min.insync.replicas=2` can no longer be met.
      * **Success Criteria:** No split-brain (only one controller active), no data loss, and no unclean leader elections.
4.  **Monitor Expected Behavior (Dual Cluster + MM2):**
      * Both clusters (`cluster-a` and `cluster-b`) remain healthy and available independently.
      * MirrorMaker 2 replication will stop and log connection errors.
      * **Success Criteria:** Local site producers/consumers are unaffected. MM2 resumes replication automatically once the partition is healed.

### 3.3 Test 3: Full Site Failure (Distant Site)

**Objective:**
Simulate a "power-down" failure of the entire distant site and execute the failover runbook.

**Procedure:**

1.  Shut down all Kubernetes nodes in the distant site (e.g., `site-b`).
2.  **Execute Runbook (Single Stretched Cluster):**
      * **If Controller Quorum is Lost (e.g., controllers were in `site-b`):** The cluster is unavailable. Declare an outage. The RTO is the time it takes to manually rebuild or re-point the controllers in the surviving site.
      * **If Controller Quorum Survives (e.g., 3-zone witness):** The cluster *should* remain available. Monitor for `UnderReplicatedPartitions`. Producers may continue if `min.insync.replicas` can be satisfied by the replicas in `site-a`.
3.  **Execute Runbook (Dual Cluster + MM2):**
      * This is the primary DR scenario for this model.
      * **Step 1:** Switch all client applications (producers and consumers) to point their `bootstrap.servers` config to the local cluster (`cluster-a-kafka-bootstrap.kafka-a.svc:9092`).
      * **Step 2:** Promote the local site (`cluster-a`) to be the primary write target.
      * **Step 3:** (On recovery) Re-enable replication, potentially in the reverse direction (B-\>A), to fail back.
      * **Success Criteria:** Clients can successfully fail over to the surviving site. RPO is minimal (only messages in-flight to MM2 that weren't replicated).