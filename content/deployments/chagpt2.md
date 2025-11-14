Below is a compact, compiled version in 3 parts: a deployment roadmap with automation guidance, plus two annexes for DR testing (local 2 VMs, and local+remote). It consolidates and streamlines everything you shared.

Part 1 — Roadmap of deployment (with automation)

1. Decide your topology (2-site constraints)
- With only two sites, a single Kafka cluster cannot simultaneously guarantee write availability and RPO≈0 during a full site loss. Pick one:
  - Option A: Single stretched cluster, controllers in one site. Simple ops, but losing controller site = outage.
  - Option B: Single stretched cluster with a third witness zone for controllers (3/5 controllers across A/B/witness). Survives loss of one site. Requires a third failure domain.
  - Option C: Two independent clusters (A and B) + MirrorMaker 2 (MM2). Best for two sites with no witness, keeps service during full site loss. Requires client failover logic.
- Recommendation for two sites without a third witness: Option C (Dual clusters + MM2).

2. SLOs and non-functionals
- Define RPO, RTO, throughput (MB/s), messages/s, and latency SLOs (p50/p95/p99).
- Cross-site RTT expectation: 0.5–5 ms metro; plan for jitter/partitions.
- Storage: site-local NVMe/block; Kafka prefers local disks.
- Security: mTLS, SASL (SCRAM/OAUTH), ACLs. DNS can route site-local clients.

3. Kubernetes/Strimzi layout essentials
- Node labels: topology.kubernetes.io/region: metro-1; topology.kubernetes.io/zone: site-a or site-b (and witness if any).
- Rack awareness: spec.kafka.rack.topologyKey = topology.kubernetes.io/zone.
- Replication:
  - default.replication.factor = 3
  - min.insync.replicas = 2 (or 3 if your latency permits)
  - unclean.leader.election.enable = false
  - offsets/topic + transaction logs RF=3, min.isr=2
- StorageClasses per site with WaitForFirstConsumer. PDBs to avoid evicting too many brokers.
- Observability: Prometheus/Grafana, Strimzi metrics, Cruise Control for rebalancing.
- Network policies: restrict and reuse for tests. External listeners per site if needed.

4. Minimal manifests (indicative; adjust to your Strimzi version)
- Single stretched KRaft with NodePools (only if you have a third witness for controllers):
  - Kafka (KRaft) CR + KafkaNodePools: controllers across A/B/witness; brokers split across A/B.
- Dual clusters (Option C) with MM2:
  - Cluster A and B (each RF=3, min.isr=2).
  - KafkaMirrorMaker2: topicsPattern/groupsPattern .*; sync.group.offsets.enabled=true; tasks.max tuned; TLS trusted certs for both clusters.

5. Performance and chaos test plan (baseline for all options)
- Workloads: small (1 KB), medium (10–100 KB), large (1–5 MB); test acks=1 vs acks=all; compression (lz4/zstd).
- Measure: p50/p95/p99 produce ack latency, end-to-end latency, throughput, ISR size, under-replicated partitions, broker/controller CPU/heap, disk I/O, cross-site bandwidth, consumer lag.
- Tools: kafka-producer/consumer-perf-test, kcat, OMB (optional), Prometheus/Grafana, Cruise Control.
- Scenarios: steady load, cross-site read/write, TLS on/off (lab only), rebalance under load, rolling upgrades under load.

6. Rollout roadmap (phases)
- Phase 0: SLOs, decision on Option A/B/C, success criteria.
- Phase 1: Local/staging labs; baseline + fault/chaos matrix; runbooks.
- Phase 2: Staging in target infra; observability; upgrade rehearsal.
- Phase 3: Client integration (multi-bootstrap, idempotent producers, retries/timeouts).
- Phase 4: Canary; soak; tune alerts.
- Phase 5: Production; ramp traffic; quarterly DR drills.

7. Key tunables and ops hygiene
- Producers: acks=all, enable.idempotence=true, delivery.timeout.ms aligned with failure detection, retry/backoff tuned for ISR churn.
- Broker: replica.lag.time.max.ms to match cross-site RTT; throttle leaders/followers during rebalances; GC, heap sizing.
- Security: mTLS, cert rotation via Strimzi CAs; ACLs; network policies.
- Alerts: UnderReplicatedPartitions > 0 (latched), OfflinePartitionsCount > 0, controller unavailability, high p95/p99 latency, GC pauses, disk > 80%.

8. How to automate deployment (IaC + GitOps + tests)
- Infra as Code:
  - Terraform/Ansible to provision K8s clusters, node pools, storage classes, LBs, network policies, DNS.
- GitOps (recommended):
  - Argo CD or Flux watches a Git repo with environment overlays (kustomize or Helm).
  - Repository structure:
    - envs/
      - lab/
      - staging/
      - prod/
    - apps/strimzi/
      - operator/ (Helm values or Kustomize)
      - clusters/
        - site-a/ (Kafka CR, KafkaNodePools, metrics, PDBs)
        - site-b/
      - mm2/ (KafkaMirrorMaker2 CR)
    - security/ (KafkaUsers, cert-manager, ExternalSecrets/SealedSecrets)
    - observability/ (Prometheus, Grafana, ServiceMonitors, alert rules)
    - chaos/ (Chaos Mesh experiments, network partition/latency YAML)
- Pipelines (CI):
  - Lint/validate YAML (kubeconform, kubeval), security scan (kics, trivy).
  - Dry-run: kind/k3d spin-up, apply manifests, smoke tests (topic create, produce/consume).
  - Optional: automated chaos stage executing a small subset of DR tests; collect metrics/artifacts.
- Promotion:
  - GitOps PRs flow env-by-env. Version-pin Strimzi/Kafka. Use automated policy checks.
- Secrets:
  - Sealed Secrets or External Secrets + a backend (Vault/SM). Never commit raw keys.
- Runbooks-as-code:
  - Makefile tasks/scripts for cluster bring-up, MM2 status, chaos start/stop, perf benchmarks, and report collection.
- Example minimal Makefile targets:
  - make bootstrap-lab (install Strimzi, monitoring)
  - make deploy-a deploy-b (clusters)
  - make deploy-mm2
  - make perf-baseline
  - make chaos-partition/chaos-latency/chaos-broker-fail
  - make report

Part 2 — Annex 1: DR tests on a local machine with 2 virtual machines

Goal
- Validate resilience and failover locally using two VMs. Support both patterns:
  - A) Single stretched K8s cluster across two VMs (for Option A/B behaviors).
  - B) Two independent clusters (for Option C with MM2).

Prereqs
- Two local VMs (Linux), each with:
  - 4+ vCPU, 8–16 GB RAM, fast disk.
  - Time sync (chrony/ntp), open firewall between VMs (broker and K8s ports).
- Pick one K8s distro: k3s, microk8s, or kubeadm. Docker/Containerd installed.
- kubectl, Helm, and (optional) Chaos Mesh. Prometheus/Grafana for metrics.

Setup A: Single stretched cluster across 2 VMs
- Install control-plane on VM1; join VM2 as worker.
- Label nodes:
  - VM1: topology.kubernetes.io/zone=site-a
  - VM2: topology.kubernetes.io/zone=site-b
- Install Strimzi operator (Helm or YAML).
- Deploy Kafka (KRaft or ZK), RF=3, min.isr=2, rack.topologyKey=topology.kubernetes.io/zone.
- Optional witness is not present here; accept trade-offs of two zones only.

Setup B: Two clusters (Option C) on 2 VMs
- Install k3s on each VM independently.
- Install Strimzi on both; deploy Kafka A on VM1, Kafka B on VM2.
- Deploy MM2 (on either VM) replicating A→B and optionally B→A. Enable offset sync.

Test data and tools
- Deploy a test producer/consumer pod (Strimzi Kafka image or kcat).
- Create topics with RF=3; generate steady load at known rates (e.g., 5–20 MB/s).

Fault injection methods (no extra tools)
- Broker crash: kubectl delete pod on a broker.
- Node loss: shutdown kubelet or reboot VM; for full site loss in Setup A, power off that VM.
- Network partition/latency on Linux:
  - Partition: sudo iptables -A INPUT -s <remote-vm-ip> -j DROP; repeat for OUTPUT.
  - Latency: sudo tc qdisc add dev <eth> root netem delay 5ms 1ms distribution normal.
  - Remove: tc qdisc del dev <eth> root; iptables -F (or delete specific rules).

DR scenarios and success criteria
- DR-01: Single broker crash
  - Expect ISR shrink then recover; no data loss with acks=all; producer retries succeed; recovery < 2–3 min.
- DR-02: Single node loss
  - As above; slightly longer recovery; no offline partitions.
- DR-03: Network latency + jitter (2–10 ms)
  - Expect modest p99 increase; no under-replicated partitions at steady state.
- DR-04: Cross-site partition between VMs
  - Option A (single cluster): produces with acks=all fail if min.isr not met; no unclean leaders; reads may continue from in-sync replicas.
  - Option C (MM2): both clusters continue locally; MM2 pauses and catches up after heal; measure RPO by counting lagged messages.
- DR-05: Full site (VM) outage
  - Option A: outage if controller site lost; otherwise, cluster may continue if min.isr satisfied; measure RTO.
  - Option C: fail clients to surviving site; MM2 later replicates back; confirm RPO ~ replication lag at moment of failure.

Metrics to capture (Grafana)
- Producer p99 latency, throughput, UnderReplicatedPartitions, OfflinePartitionsCount, consumer lag, broker/controller CPU/mem, disk I/O, MM2 lag.

Minimal commands (illustrative)
- Topic create (inside a Kafka tools pod):
  - kafka-topics.sh --bootstrap-server <bootstrap>:9092 --create --topic t1 --partitions 12 --replication-factor 3
- Producer perf:
  - kafka-producer-perf-test.sh --topic t1 --num-records 2_000_000 --record-size 1000 --throughput -1 --producer-props bootstrap.servers=<bootstrap>:9092 acks=all linger.ms=5 batch.size=131072 compression.type=lz4
- Kill broker:
  - kubectl -n kafka delete pod -l strimzi.io/name=<cluster>-kafka --limit=1 --force --grace-period=0
- Add latency:
  - tc qdisc add dev eth0 root netem delay 5ms 1ms
- Partition link:
  - iptables -I INPUT -s <peer-ip> -j DROP; iptables -I OUTPUT -d <peer-ip> -j DROP

Pass/fail gates
- No unclean leader election.
- No data loss with acks=all.
- p99 produce latency within agreed SLO when healthy; bounded spikes under faults.
- Recovery times within RTO; replication catch-up within acceptable window.

Part 3 — Annex 2: DR tests with one local machine and another distant machine

Goal
- Validate DR when one site is truly remote (WAN/Internet). Focus on MM2-based dual clusters (Option C) to maintain availability across sites.

Prereqs and network
- One local K8s cluster (kind/k3d/k3s) and one remote K8s cluster (cloud or on-prem).
- Stable IP reachability between clusters:
  - Recommended: WireGuard or IPSec site-to-site tunnel to get flat, private connectivity.
  - Alternatively: expose remote bootstrap via public LB with TLS/mTLS and firewall rules.
- Time sync (NTP) and TLS in place. Ensure CA trust exchange between clusters for MM2.

Deploy
- Install Strimzi in both clusters; deploy Kafka A (local) and Kafka B (remote). RF=3, min.isr=2 in each site.
- Deploy MM2 close to the target (or source); set:
  - topicsPattern/groupsPattern=".*"
  - sync.group.offsets.enabled=true
  - heartbeat/checkpoint connectors enabled
  - tasks.max tuned for bandwidth and CPU
  - TLS trustedCertificates for both A and B
- Verify topic replication and offset checkpointing.

WAN-specific tests
- WAN latency baseline:
  - Measure RTT and jitter across tunnel or LB. Record baseline p95/p99 produce latency and MM2 lag at steady load.
- DR-10: WAN latency increases (add 20–50 ms with tc on one side)
  - Expect increased p99 produce latency (if cross-site writes) and higher MM2 lag; confirm no data loss; lag returns to baseline after removal.
- DR-11: Packet loss 1–3%
  - tc qdisc add dev eth0 root netem loss 2%; replication slows; MM2 retries; confirm durable catch-up.
- DR-12: Link partition (drop tunnel or block LB SG/ACL)
  - MM2 pauses; local cluster continues serving; on heal, MM2 catches up; measure RPO ~ MM2 lag at failure; confirm ordering per partition.
- DR-13: Remote site outage
  - Power down remote cluster or scale brokers/statefulset to 0; local site continues; upon recovery, monitor MM2 catch-up throughput and time.
- DR-14: Failover clients to remote
  - Switch client bootstrap to remote cluster B; confirm topic pre-provisioning and ACL parity; validate read-your-writes semantics with new writes on B.
- DR-15: Failback
  - Reverse MM2 or enable B→A; when caught up, move producers back to A; ensure consumers re-aligned via MM2 checkpoints or group resets.

Operational considerations over WAN
- Security: mTLS everywhere; rotate certs; principle of least privilege on LB/firewall.
- Throughput tuning:
  - Producer: linger.ms, batch.size, compression (lz4/zstd); acks=all for durability.
  - MM2: tasks.max, max.in.flight.requests.per.connection, fetch/produce batch sizes, replication.factor for destination topics.
- Observability:
  - Track MM2 consumer lag and task failures; export metrics to Prometheus; alert when lag > threshold (e.g., >5 s or app-defined).
- Capacity and SLOs:
  - Validate that WAN bandwidth × window > peak replication needs; add smoothing (linger/batching) to reduce small-message overhead.
  - Set explicit RPO expectation based on observed MM2 lag under peak.

Automation (local+remote)
- GitOps in both sites; Argo CD applications per site with environment overlays.
- A “DR test” pipeline:
  - Stage 1: deploy updates, validate health and baselines.
  - Stage 2: run WAN chaos (tc/iptables or cloud NACL flips) via remote runners.
  - Stage 3: collect Prometheus snapshots and Kafka metrics; publish a short HTML/Markdown report with pass/fail.
- Secrets management across sites via External Secrets with separate backends (Vault/SM) and per-site roles.

Quick reference commands (WAN tests)
- Add WAN latency on local egress:
  - tc qdisc add dev eth0 root netem delay 30ms 5ms
- Add packet loss:
  - tc qdisc change dev eth0 root netem loss 2%
- Drop tunnel:
  - systemctl stop wg-quick@wg0 (or down the interface); or block LB security group temporarily.
- Produce/consume sanity:
  - kcat -b <bootstrap>:9092 -t t1 -P/-C; or kafka-producer-perf-test.sh and kafka-consumer-perf-test.sh.

Closing notes
- Add a lightweight third witness domain, a single-cluster KRaft controller quorum across A/B/witness (Option B) can maintain availability through a full site loss. Otherwise, dual clusters with MM2 (Option C) is the pragmatic and robust approach for two sites.
- Keep DR drills automated and frequent. Measure and publish RTO/RPO from every drill.