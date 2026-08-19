# NetworkPolicy

Default-deny per namespace, with an explicit allow-list. The second layer under
`../mtls/` and the AuthorizationPolicies that follow it: mTLS proves *who* a
workload is, NetworkPolicy limits *what it can reach at all*, and the two are
deliberately independent so that a failure in one does not open the other.

Part of #9.

## Enforcement is currently off

The AWS VPC CNI node agent runs with `--enable-network-policy=false`. Confirmed
18 August 2026:

```
kubectl get ds aws-node -n kube-system -o yaml | grep -i network-policy
```

So NetworkPolicy objects are accepted by the API server and never enforced.
Nothing rejects them, nothing warns, and `kubectl get networkpolicy` shows them
present and apparently working.

That is the worst failure mode for a security control: it looks deployed. Anyone
auditing this namespace would see a default-deny and reasonably conclude the
namespace is isolated.

Enabling enforcement is a one-line change to the `aws-vpc-cni` HelmRelease in
`gitops-flux`, and belongs to `@infra`. The value is not set in either the base
or the dev overlay, so this is the chart default rather than a decision anyone
made.

## Sequencing

**Enabling enforcement while no policies exist changes nothing** — the
Kubernetes default is allow-all, and a namespace with no policy stays that way.

**Enabling it after these land starts denying traffic immediately.**

So the order is not symmetric, and whoever flips the flag needs to know which
namespaces already carry a default-deny. Today that is `cert-manager` only. Flip
the flag, then watch cert-manager before adding more.

## Rollout

Same shape as the mTLS rollout in `../README.md`, and the Kyverno lifecycle in
`../../kyverno/policies/README.md` — nothing goes straight to its final state.

```
write default-deny + allow-list for one namespace
  → merge (inert while enforcement is off)
  → enable enforcement
  → watch that namespace for a sprint
  → next namespace
```

One namespace per pull request. The blast radius of a wrong allow-list is the
whole namespace, and the symptoms do not name a NetworkPolicy.

## Namespaces

| Namespace | Status | Why |
|---|---|---|
| `cert-manager` | written | Small, enumerable traffic, nothing else calls it |
| `monitoring` | not yet | Prometheus initiates connections into every namespace; needs its own thinking |
| `teleport` | not yet | New, and the access path everyone will depend on |
| `platform-istio` | excluded | Ingress gateway. Front door, handled last |
| `flux-system` | excluded | Manages the mesh and everything else. Locking it out is a bootstrapping problem |
| `karpenter` | excluded | Must keep working during an outage or you cannot scale out of it |
| `sealed-secrets` | excluded | Recovery path must not depend on the network layer being right |
| `kube-system` | excluded | Control plane components; several cannot take a policy. Needs its own decision |
| `default`, `kube-node-lease`, `kube-public` | excluded | No workloads |

## Two rules every namespace needs

Both were found by reading the running pods rather than by writing a template,
and both fail quietly.

### Liveness probes

The kubelet calls probes from the node address. It is not a pod, so no
`podSelector` matches it, and a default-deny blocks it. The pod then fails its
probe and restart-loops — which reads as an application fault, not a network
policy.

Probe ports are frequently **named** rather than numeric in the pod spec, so
they do not appear in a casual `kubectl get`. Resolve them before writing the
policy:

```
kubectl get pods -n <ns> -o custom-columns='NAME:.metadata.name,PROBE:.spec.containers[*].livenessProbe.httpGet.port'
```

In `cert-manager` this caught two pods: the webhook (`healthcheck`, 6080) and
the controller (`http-healthz`, 9403). A policy allowing only the admission
webhook port would have restart-looped both.

Note also that policy ports are **container** ports. The `cert-manager-webhook`
Service publishes 443 and maps it to container port 10250; 443 in a policy would
be wrong.

### Prometheus scraping

Prometheus initiates the connection, so the rule is **ingress to the scraped
namespace from `monitoring`** — not egress from Prometheus.

Miss it and metrics stop arriving with no error and no restart. A target goes
stale, and alerts stop firing because the series they depend on no longer
exists. The alert that should tell you something is wrong is itself the thing
that broke.

Every namespace that gets a default-deny needs this rule in the same pull
request.

The namespace was renamed from `lab-observability` to `monitoring` on
14 August 2026.

## What this layer does not do

NetworkPolicy selects by pod, namespace and CIDR. It cannot select by hostname.

`cert-manager` needs Route 53, STS, the Let's Encrypt ACME directory and public
authoritative DNS — none of which have stable addresses that can be written
here. The egress rule is therefore `0.0.0.0/0` on 443 with RFC1918 and the
instance metadata address carved out.

That does not restrict where cert-manager can reach on the internet, and the
policy should not be read as if it does. What it restricts is **lateral
movement**: a compromised cert-manager pod cannot reach another namespace's
services. That is the threat this layer exists for. Filtering egress by
destination name needs a proxy or a CNI that resolves names, and neither is in
scope.