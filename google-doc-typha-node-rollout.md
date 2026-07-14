# Calico Operator: Typha / calico-node Concurrent Rollout During Upgrades

**Author:** Adheip Singh
**Date:** 2026-07-03
**Context:** Customer upgrade Calico 3.31.3 -> 3.32.0, ~200-node cluster, calico-node NotReady flaps

---

## 1. Summary

During a version upgrade on a ~200-node cluster, calico-node pods flapped to `NotReady` while Typha was also rolling. New nodes were joining mid-upgrade. We traced the behavior to how the operator applies both components and documented our findings below.

---

## 2. How the Operator Applies Components During an Upgrade

```
 1. New operator starts with updated image refs
 2. Installation controller reconciles
 3. Renders Typha component                -> core_controller.go:1483
 4. Renders Node component                 -> core_controller.go:1651
 5. Applies all components in a for loop   -> core_controller.go:1755-1760
    - Patches Typha Deployment spec
    - Patches calico-node DaemonSet spec       <- milliseconds later
 6. Kubernetes rolls both concurrently — no coordination between them
```

---

## 3. No Readiness Gating Between Components

| Check | Present? | Reference |
|-------|----------|-----------|
| Typha `Ready()` | Returns `true` unconditionally | `typha.go:167` |
| Node `Ready()` | Returns `true` unconditionally | `node.go:271` |
| Handler checks `Ready()` before applying | Yes, but no-op since both return `true` | `component.go:447` |
| Requeue-until-Typha-available | Does not exist | — |
| Deployment Available condition check | Does not exist | — |

Namespace migration (`namespace_migration.go:298-312`) does have explicit Typha-first ordering — this pattern was not carried into the main reconcile path.

---

## 4. Dependency Chain

```
calico-node Ready
    -> Felix Ready          (readiness probe: --felix-ready, node.go:1795)
    -> Felix synced with Typha   (implicit, not probed directly)
    -> Typha pod healthy
```

---

## 5. Typha Autoscaler

- Polls every **10 seconds** (`typha_autoscaler.go:40`)
- Formula: `(nodes/200) + 2`, min 3 for 5+ nodes (`autoscale.go:32-56`)
- Not event-driven — node joins are detected on next tick
- `triggerRun()` only invoked when autoscaler is already degraded (`core_controller.go:1007`)

---

## 6. Hardcoded vs Configurable

### Typha Deployment

| Setting | Default | Configurable | CRD Path |
|---------|---------|--------------|----------|
| MaxSurge | `100%` | Yes | `spec.typhaDeployment.spec.strategy.rollingUpdate.maxSurge` |
| MaxUnavailable | `1` | Yes | `spec.typhaDeployment.spec.strategy.rollingUpdate.maxUnavailable` |
| terminationGracePeriodSeconds | `300` | Yes | `spec.typhaDeployment.spec.template.spec.terminationGracePeriodSeconds` |
| minReadySeconds | `0` | Yes | `spec.typhaDeployment.spec.minReadySeconds` |
| Probe handler | Set by operator | No | Timing only via `.readinessProbe` |
| PDB | minAvailable: 1 | Yes | `spec.typhaPodDisruptionBudget` |
| Replica count | Auto-scaled | No | No CRD field |

### calico-node DaemonSet

| Setting | Default | Configurable | CRD Path |
|---------|---------|--------------|----------|
| MaxUnavailable | `1` | Yes | `spec.nodeUpdateStrategy.rollingUpdate.maxUnavailable` |
| terminationGracePeriodSeconds | `5` | No | Hardcoded `node.go:62` |
| minReadySeconds | `0` | Yes | `spec.calicoNodeDaemonSet.spec.minReadySeconds` |
| Probe handler | Set by operator (`--felix-ready`) | No | Timing only via `.readinessProbe` |

### Operator-Level

| Setting | Default | Configurable |
|---------|---------|--------------|
| Rollout ordering | None — both applied in same pass | No |
| Autoscaler poll interval | `10s` | No |
| Nodes-per-Typha ratio | `200` | No |
| Component `Ready()` gating | Always `true` | No |

---

## 7. Source References

| What | File | Lines |
|------|------|-------|
| Typha component appended | `pkg/controller/installation/core_controller.go` | 1483 |
| Node component appended | same | 1651 |
| Component apply loop | same | 1755-1760 |
| Autoscaler degraded check | same | 1007-1012 |
| Node update strategy defaults | same | 749-759 |
| Typha `Ready()` | `pkg/render/typha.go` | 167 |
| Typha update strategy and rationale | same | 394-408 |
| Typha termination grace period | same | 58, 394 |
| Node `Ready()` | `pkg/render/node.go` | 271 |
| Node termination grace period | same | 62 |
| Node readiness probe | same | 1789-1823 |
| Felix Typha env vars | same | 1531-1535 |
| Autoscaler poll interval | `pkg/controller/installation/typha_autoscaler.go` | 40 |
| Autoscaler main loop | same | 163-196 |
| Autoscale formula | `pkg/common/autoscale.go` | 32-56 |
| Handler `Ready()` check | `pkg/controller/utils/component.go` | 447-450 |
| Namespace migration ordering | `pkg/controller/migration/namespace_migration.go` | 298-312 |
| Typha CRD overrides | `api/v1/typha_deployment_types.go` | 131-170 |
| Node CRD overrides | `api/v1/calico_node_types.go` | 63-146 |
