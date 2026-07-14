# Root Cause Analysis: calico-node NotReady During Upgrade

**Author:** Adheip Singh
**Date:** 2026-07-14
**Context:** Customer upgrade Calico 3.31.3 -> 3.32.0, ~200-node cluster, calico-node NotReady flaps
**Reproduction:** 30-node cluster (1 master + 29 workers), eu-central-1, t3.medium spot instances, kubeadm v1.32.13

---

## 1. Summary

During a version upgrade, calico-node pods flap to NotReady across the entire cluster. The root cause is **Felix losing its Typha connection during the concurrent Typha rollout**, which causes old calico-node pods to fail their readiness probe. After 3 consecutive probe failures (90 seconds), the Kubernetes DaemonSet controller classifies old pods as "unavailable" and **bypasses the maxSurge limit**, replacing all pods simultaneously. This collapses the BGP full mesh, causing an avalanche of NotReady.

**The root cause is NOT image pull latency and NOT BGP peering. It is Felix -> Typha reconnection delay during concurrent component rollout.**

---

## 2. Reproduction Setup

- **Cluster:** 30 nodes (1 master + 29 workers), kubeadm v1.32.13
- **From version:** Calico v3.31.3 (operator v1.40.3)
- **To version:** Calico v3.32.1 (operator v1.42.3)
- **calico-node DaemonSet:** maxSurge=4, maxUnavailable=0
- **Typha:** 3 replicas (autoscaler: `(nodes/200) + 2`, min 3)
- **Typha resources:** requests 100m/128Mi, limits 500m/256Mi
- **Typha rollout strategy:** MaxSurge=100%, MaxUnavailable=1, terminationGracePeriod=300s

---

## 3. Reproduction Results

### 3.1 DaemonSet Monitoring (polled every 2 seconds)

```
Time     | calico-node Ready/Desired | Updated | Typha  | Event
---------|---------------------------|---------|--------|------
23:47:36 | 30/30                     | 0       | 3/3    | Operator patches both Typha + calico-node specs
23:48:53 | 30/30                     | 1       | 3/3    | First calico-node pod updating
23:48:56 | 28/30                     | 3       | 3/3    | *** DISRUPTION STARTS ***
23:48:59 | 21/30                     | 10      | 3/3    | Rapid cascade - 9 pods NotReady
23:49:02 | 20/30                     | 11      | 3/3    | Continuing drop
23:49:24 | 18/30                     | 12      | 3/3    | Dip to 18
23:49:27 | 5/30                      | 27      | 3/3    | CATASTROPHIC - only 5 pods Ready
23:49:33 | 4/30                      | 30      | 3/3    | NADIR - 26 pods NotReady simultaneously
23:49:46 | 9/30                      | 30      | 3/3    | Recovery starts
23:50:15 | 26/30                     | 30      | 3/3    | Most recovered
23:50:20 | 30/30                     | 30      | 3/3    | Fully stable
```

**Total disruption duration: ~84 seconds. Nadir: only 4/30 calico-node pods Ready.**

Note: Typha stayed 3/3 throughout. The disruption is entirely in calico-node.

---

## 4. Root Cause Analysis

### 4.1 The Primary Failure: Felix 503 (NOT BIRD, NOT image pull)

The readiness probe checks BOTH `-bird-ready` AND `-felix-ready`. The **dominant failure mode is Felix reporting 503**, even when BIRD has full BGP peering:

```
23:48:27 | BGP peering established = 29   <-- ALL 29 peers up, BIRD is fine
           calico/node is not ready: felix is not ready: readiness probe reporting 503
```

Felix 503 began at **23:47:53** -- this is **60 seconds BEFORE any calico-node pod was replaced** (first pod updated at 23:48:53). This proves old v3.31.3 calico-node pods went NotReady while still running, due to losing their Typha connection during the Typha rollout.

### 4.2 The Cascade Mechanism

```
Step 1: Operator patches Typha + calico-node specs in same reconcile loop
        (core_controller.go:1755-1760, milliseconds apart)

Step 2: Typha rolls: new ReplicaSet 0->3 (MaxSurge=100%), old RS 3->2->1->0 (~5s)

Step 3: Old Typha pods enter Terminating -> shed Felix connections
        Felix on ALL 30 nodes loses Typha connection -> reports 503 (not in-sync)

Step 4: Readiness probe counts failures (periodSeconds=30, failureThreshold=3):
        23:47:53  Felix 503  -> probe fail #1
        23:48:23  Felix 503  -> probe fail #2
        23:48:53  Felix 503  -> probe fail #3  -> POD TRANSITIONS TO NotReady
                                                  ^^ exactly when disruption starts!

Step 5: DaemonSet controller sees old pods are "unavailable"
        Per Kubernetes docs:
        "If the old pod becomes unavailable for any reason (Ready transitions to
         false, is evicted, or is drained) an updated pod is immediately created
         on that node WITHOUT CONSIDERING SURGE LIMITS."
        -> maxSurge=4 is BYPASSED -- ALL nodes get replacement pods simultaneously

Step 6: 30 BIRD processes restart at once -> full-mesh BGP collapses
        Peer counts drop: 29 -> 26 -> 22 -> 15 -> 5 -> 0
        Only 4/30 pods Ready at nadir
```

### 4.3 Why maxSurge=1 Works Fine

The same Felix 503 cascade happens with maxSurge=1. However, with maxSurge=1, the DaemonSet controller starts replacing 1 pod before the 90-second failureThreshold triggers. By the time old pods transition to NotReady, some have already been replaced and recovered. The BGP mesh never loses enough peers to cascade catastrophically.

### 4.4 Why maxSurge=4 Breaks

With maxSurge=4, the DaemonSet controller starts 4 replacements. But before those complete, the 90-second failureThreshold fires for Felix 503 on ALL remaining old pods. The DaemonSet controller then bypasses maxSurge for all "unavailable" pods, triggering simultaneous replacement across the cluster. The BGP mesh collapses.

---

## 5. Evidence: Readiness Probe Failure Logs

### 5.1 Phase 1: Felix 503 with full BGP mesh (23:47:53 - 23:48:57)

Old v3.31.3 pods losing Typha connection. BIRD is healthy, Felix is the problem:

```
Readiness probe failed: 2026-07-13 23:47:53.314 [INFO][32] node/health.go 207:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:24.318 [INFO][380] node/health.go 207:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:25.709 [INFO][852] node/health.go 206:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:27.324 [INFO][813] node/health.go 206:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:54.313 [INFO][466] node/health.go 207:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:54.492 [INFO][877] node/health.go 206:
  Number of node(s) with BGP peering established = 29
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:48:57.326 [INFO][838] node/health.go 206:
  Number of node(s) with BGP peering established = 26
  calico/node is not ready: felix is not ready: readiness probe reporting 503
```

### 5.2 Phase 2: BGP mesh collapsing (23:49:00 - 23:49:32)

As calico-node pods restart simultaneously, BGP peers drop across all nodes:

```
Readiness probe failed: 2026-07-13 23:49:00.092 [INFO][876] node/health.go 206:
  Number of node(s) with BGP peering established = 22
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:49:14.172 [INFO][501] node/health.go 207:
  Number of node(s) with BGP peering established = 22
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:49:25.706 [INFO][926] node/health.go 206:
  Number of node(s) with BGP peering established = 24
  calico/node is not ready: BIRD is not ready: BGP not established with
  172.31.32.110,172.31.32.169,172.31.35.202,172.31.43.51,172.31.46.221

Readiness probe failed: 2026-07-13 23:49:26.600 [INFO][880] node/health.go 206:
  Number of node(s) with BGP peering established = 19
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:49:26.711 [INFO][859] node/health.go 206:
  Number of node(s) with BGP peering established = 18
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:49:27.723 [INFO][854] node/health.go 206:
  Number of node(s) with BGP peering established = 15
  calico/node is not ready: felix is not ready: readiness probe reporting 503

Readiness probe failed: 2026-07-13 23:49:31.709 [INFO][848] node/health.go 206:
  Number of node(s) with BGP peering established = 15
  calico/node is not ready: felix is not ready: readiness probe reporting 503
```

### 5.3 Phase 3: New pods initializing (23:49:50+)

Fresh v3.32.1 pods starting, BIRD not yet running:

```
Readiness probe failed: calico/node is not ready: BIRD is not ready:
  error querying BIRD: unable to connect to BIRDv4 socket:
  dial unix /var/run/bird/bird.ctl: connect: no such file or directory

Readiness probe failed: calico/node is not ready: BIRD is not ready:
  failed to stat() nodename file: stat /var/lib/calico/nodename: no such file or directory
```

### 5.4 Complete BGP mesh loss on one node

One pod lost ALL 22+ BGP peers simultaneously:

```
Readiness probe failed: 2026-07-13 23:48:59.148 [INFO][488] node/health.go 207:
  Number of node(s) with BGP peering established = 0
  calico/node is not ready: BIRD is not ready: BGP not established with
  172.31.32.110,172.31.32.169,172.31.33.188,172.31.33.247,172.31.34.233,
  172.31.35.202,172.31.35.52,172.31.36.55,172.31.37.188,172.31.38.182,
  172.31.38.255,172.31.38.52,172.31.38.70,172.31.39.121,172.31.39.140,
  172.31.39.17,172.31.39.91,172.31.42.181,172.31.42.245,172.31.42.37,
  172.31.43.251,172.31.43.51
```

---

## 6. Kubernetes Events

### 6.1 Typha Deployment Events

Confirms concurrent Typha rollout (new RS created, old RS scaled down within seconds):

```
5m38s  Normal  ScalingReplicaSet  deployment/calico-typha  Scaled up replica set calico-typha-594b8f658d from 0 to 3
5m38s  Normal  ScalingReplicaSet  deployment/calico-typha  Scaled down replica set calico-typha-59879fb4b from 3 to 2
5m34s  Normal  ScalingReplicaSet  deployment/calico-typha  Scaled down replica set calico-typha-59879fb4b from 2 to 1
5m33s  Normal  ScalingReplicaSet  deployment/calico-typha  Scaled down replica set calico-typha-59879fb4b from 1 to 0
```

### 6.2 Liveness Probe Failure

Felix liveness probe also failed during the disruption:

```
Liveness probe failed: Get "http://localhost:9099/liveness": dial tcp 127.0.0.1:9099: connect: connection refused
```

---

## 7. Readiness Probe Configuration

```json
{
    "exec": {
        "command": ["/bin/calico-node", "-bird-ready", "-felix-ready"]
    },
    "failureThreshold": 3,
    "periodSeconds": 30,
    "successThreshold": 1,
    "timeoutSeconds": 10
}
```

- Both `-bird-ready` AND `-felix-ready` must pass
- 3 consecutive failures (3 x 30s = **90 seconds**) to transition to NotReady
- This 90-second window perfectly matches the observed delay between Typha rollout (23:47:36) and calico-node disruption (23:48:56)

---

## 8. Cluster Configuration at Time of Test

### calico-node DaemonSet Strategy
```json
{"rollingUpdate":{"maxSurge":4,"maxUnavailable":0},"type":"RollingUpdate"}
```

### Typha Deployment Strategy (set by operator)
- MaxSurge: 100% (start ALL new Typhas first)
- MaxUnavailable: 1 (kill old ones slowly)
- terminationGracePeriodSeconds: 300 (5 min graceful shed)

### calico-node Environment Variables
```
DATASTORE_TYPE=kubernetes
CALICO_NETWORKING_BACKEND=bird
FELIX_TYPHAK8SNAMESPACE=calico-system
FELIX_TYPHAK8SSERVICENAME=calico-typha
FELIX_HEALTHENABLED=true
FELIX_HEALTHPORT=9099
```

### Typha Resources
```json
{"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

### calico-node uses hostNetwork
```
hostNetwork: true
```

### BIRD listens on host
```
LISTEN 0  8  0.0.0.0:179  0.0.0.0:*   (BGP)
LISTEN 0  4096  127.0.0.1:9099  0.0.0.0:*   (Felix health)
```

---

## 9. Operator Code: No Readiness Gating Between Components

```
core_controller.go:1483  -> Renders Typha component
core_controller.go:1651  -> Renders Node component
core_controller.go:1755-1760 -> Applies ALL components in a for loop
                                - Patches Typha Deployment spec
                                - Patches calico-node DaemonSet spec  <- milliseconds later
                                Kubernetes rolls both concurrently
```

- `typha.go:167`: `Ready()` returns `true` unconditionally
- `node.go:271`: `Ready()` returns `true` unconditionally
- `component.go:447`: Handler checks `Ready()` before applying, but it's a no-op since both return `true`

---

## 10. Fix: Typha Ready Gating

### Branch
- `fix/typha-ready-gating` (master)
- `fix/typha-ready-gating-v1.42` (release-v1.42)
- Image: `deepbi/tigera-operator:v1.42.4-typha-ready-gating`

### Changes

**`pkg/render/node.go`**: Gate calico-node rollout on Typha availability:
```go
func (c *nodeComponent) Ready() bool {
    return c.cfg.TyphaRolledOut  // was: return true
}
```

**`pkg/controller/installation/core_controller.go`**: Check Typha Deployment status:
```go
typhaRolledOut := false
typhaDeployment := &appsv1.Deployment{}
typhaKey := types.NamespacedName{Name: common.TyphaDeploymentName, Namespace: common.CalicoNamespace}
if err := r.client.Get(ctx, typhaKey, typhaDeployment); err != nil {
    if !apierrors.IsNotFound(err) {
        r.status.SetDegraded(operatorv1.ResourceReadError, "Unable to read Typha Deployment", err, reqLogger)
        return reconcile.Result{}, err
    }
    typhaRolledOut = true  // First install, no Typha yet
} else if typhaDeployment.Spec.Replicas != nil {
    typhaRolledOut = typhaDeployment.Status.ObservedGeneration >= typhaDeployment.Generation &&
        typhaDeployment.Status.UpdatedReplicas == *typhaDeployment.Spec.Replicas &&
        typhaDeployment.Status.AvailableReplicas == *typhaDeployment.Spec.Replicas
}
```

### How the Fix Addresses the Root Cause

```
Without fix:                          With fix:
1. Patch Typha spec                   1. Patch Typha spec
2. Patch calico-node spec (ms later)  2. Node Ready() returns false
3. Both roll concurrently             3. Handler skips calico-node: "Component is not ready"
4. Felix loses Typha -> 503           4. Typha rolls, Felix reconnects
5. failureThreshold fires -> NotReady 5. Felix recovers, pods healthy again
6. DaemonSet bypasses maxSurge        6. Operator re-reconciles, Typha now rolled out
7. BGP mesh collapses                 7. Node Ready() returns true
8. 4/30 Ready at nadir                8. calico-node DaemonSet updated
                                      9. maxSurge=4 respected (all old pods healthy)
                                      10. Rolling update proceeds normally
```

---

## 11. Typha Autoscaler

| Nodes | Typhas | Felix/Typha |
|-------|--------|-------------|
| 30    | 3      | ~10         |
| 200   | 3      | ~67         |
| 500   | 4      | ~125        |
| 1000  | 7      | ~143        |

Formula: `(nodes/200) + 2`, min 3 for 5+ nodes (`pkg/common/autoscale.go:32-56`)

---

## 12. Key Kubernetes Behavior

From the [Kubernetes DaemonSet documentation](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/):

> If the old pod becomes unavailable for any reason (Ready transitions to false, is evicted, or is drained) an **updated pod is immediately created on that node without considering surge limits**.

This is the critical mechanism that turns a controlled maxSurge=4 rollout into a cluster-wide avalanche when old pods go NotReady from Felix 503.
