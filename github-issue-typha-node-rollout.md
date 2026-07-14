# Title

During version upgrades, Typha and calico-node roll concurrently — is readiness gating between them intentional?

# Body

## Context

During a Calico 3.31.3 to 3.32.0 upgrade on a ~200-node cluster, we observed calico-node pods flapping to `NotReady` while Typha was also rolling. New nodes were joining the cluster mid-upgrade via autoscaling. We traced this to how the operator applies both components and wanted to understand whether the current behavior is by design.

## What We See in the Code

### Both components are applied in the same reconcile pass

The Installation controller appends both Typha and Node to the same `components` slice and applies them in a sequential loop:

```go
// core_controller.go:1483
components = append(components, render.Typha(&typhaCfg))

// core_controller.go:1651
components = append(components, render.Node(&nodeCfg))

// core_controller.go:1755-1760
for _, component := range components {
    if err := handler.CreateOrUpdateOrDelete(ctx, component, nil); err != nil {
        r.status.SetDegraded(operatorv1.ResourceUpdateError, "Error creating / updating resource", err, reqLogger)
        return reconcile.Result{}, err
    }
}
```

Both Deployment and DaemonSet specs are patched within milliseconds of each other. Kubernetes then drives both rollouts concurrently.

### Both `Ready()` methods return `true` unconditionally

```go
// typha.go:167
func (c *typhaComponent) Ready() bool { return true }

// node.go:271
func (c *nodeComponent) Ready() bool { return true }
```

The handler at `component.go:447` checks `Ready()` before applying, but since both always return `true`, no gating occurs.

### calico-node readiness transitively depends on Typha

The readiness probe checks `--felix-ready` (`node.go:1795`), and Felix cannot report ready until it syncs with Typha. During the overlap window where old Typhas are terminating and new Typhas may not yet be serving, restarted calico-node pods can fail to find a healthy Typha.

### Typha autoscaler polls every 10 seconds

The autoscaler uses a ticker (`typha_autoscaler.go:40`, `defaultTyphaAutoscalerSyncPeriod = 10 * time.Second`), not event-driven node watches. When nodes join mid-upgrade, there's a delay before Typha scales up.

### Namespace migration already has explicit ordering

In `namespace_migration.go:298-312`, the operator waits for Typha readiness before migrating calico-node. This ordering doesn't exist in the main reconcile path.

## Hardcoded vs Configurable

Some settings relevant to upgrade behavior are not exposed:

| Setting | Default | Configurable? |
|---------|---------|---------------|
| Typha `MaxSurge` | `100%` | Yes |
| Typha `MaxUnavailable` | `1` | Yes |
| Typha `terminationGracePeriodSeconds` | `300` | Yes |
| calico-node `MaxUnavailable` | `1` | Yes |
| calico-node `terminationGracePeriodSeconds` | `5` | No (`node.go:62`) |
| Autoscaler poll interval | `10s` | No (`typha_autoscaler.go:40`) |
| Nodes-per-Typha ratio | `200` | No (`autoscale.go:33`) |

## Related Issues

- #3095 — Requests configurable autoscaler profile
- #1295 — Requests manual Typha replica override
- #1441 — Autoscaler should exclude tainted nodes
- projectcalico/calico#6336 — Version skew during 3.22->3.23 upgrade
- projectcalico/calico#5211 — Typha stuck triggers calico-node route removal

## Questions

1. Is the concurrent rollout of Typha and calico-node intentional, or would the operator benefit from gating calico-node behind Typha readiness (similar to what namespace migration already does)?

2. Would a change to `nodeComponent.Ready()` to check Typha Deployment availability be a reasonable approach? The handler already skips components where `Ready()` returns `false` and the controller requeues naturally.

3. Should the Typha autoscaler react to node-join events immediately rather than polling every 10 seconds? The node informer is already available in the autoscaler struct.

4. Is there a reason calico-node's `terminationGracePeriodSeconds` (5s) is not exposed as a CRD override while Typha's (300s) is?

## Environment

- **Calico:** 3.31.3 -> 3.32.0
- **Operator:** v1.40.x -> v1.42.x
- **Cluster:** ~200 nodes, new nodes joining mid-upgrade
