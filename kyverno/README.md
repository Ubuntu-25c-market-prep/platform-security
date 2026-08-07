# Kyverno — HA install

`values.yaml` in this directory configures a highly-available install of the
Kyverno Helm chart (`kyverno/kyverno`). Chart defaults are used for everything
else; version/image tags are set on the Helm release command, not in this file.

## What values.yaml sets, and why

| Controller | Replicas | PDB `minAvailable` | Notes |
|---|---|---|---|
| `admissionController` | 3 | 2 | Also sets `topologySpreadConstraints` (spread by `kubernetes.io/hostname`) so replicas aren't packed onto one node, and resource requests/limits (100m/128Mi request, 384Mi memory limit, no CPU limit) |
| `backgroundController` | 2 | 1 | Leader election is automatic at replicas > 1 — no extra config |
| `reportsController` | 2 | 1 | Same leader-election behavior as backgroundController |
| `cleanupController` | 2 | 1 | Lower resource footprint — not on the live admission path |

Default `helm install kyverno kyverno/kyverno` runs a single replica of each
controller. That's fine for a throwaway dev cluster, but not for something
that admission-controls a real shared cluster: if the single
admission-controller pod is mid-restart during a deploy, every apply in the
cluster either hangs or fails, depending on `failurePolicy`.

Each PodDisruptionBudget exists so a voluntary disruption (node drain,
cluster upgrade) can't take a controller below its `minAvailable` count —
without it, a rolling node upgrade could legally evict enough replicas at
once to leave a single point of failure during the exact window the cluster
is already being disrupted.

CPU limits are deliberately omitted: limiting CPU on a Go binary can cause
throttling under load, which is the wrong failure mode for something sitting
in the admission path. Memory is capped so a leak/spike gets OOM-killed and
restarted instead of pressuring the node.

## Applying

```
helm upgrade --install kyverno kyverno/kyverno -n kyverno -f kyverno/values.yaml
```

## Verification

Tested on a local `kind` cluster (`security-lab-control-plane`) by draining
a node running Kyverno pods:

```
kubectl drain security-lab-control-plane --ignore-daemonsets --delete-emptydir-data --timeout=60s
```

The drain was blocked with:

```
Cannot evict pod as it would violate the pod's disruption budget.
```

This confirms the PodDisruptionBudgets are enforced: Kubernetes refused a
voluntary eviction that would have dropped a controller below its configured
`minAvailable`.
