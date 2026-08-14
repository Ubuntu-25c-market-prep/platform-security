# zerotrust

STRICT mTLS, AuthorizationPolicy and SPIFFE workload identity. Wave 7.

Owned by `@zerotrust`. Everything here depends on Istio, and on namespaces
actually being in the mesh — see below, because that second condition is not yet
met and it is the one that catches people out.

## Current state, 14 August 2026

| | |
|---|---|
| istiod | running, `platform-istio`, revision `1-30-3` |
| Ingress gateway | running, behind an NLB |
| Root namespace | **`platform-istio`**, not `istio-system` |
| Namespaces with sidecar injection | **none** |
| PeerAuthentication objects | none |

The Istio CRDs are registered, so the API server will accept a
`PeerAuthentication` today. That is exactly why this needs care: the manifest
applies cleanly and does nothing, right up until the moment it does something
unintended.

## Why mTLS cannot simply be switched on

**mTLS is enforced by the sidecar.** A workload with no Envoy sidecar has nothing
to enforce with. With no namespace labelled for injection, a STRICT policy is
inert across the whole platform — with one exception, and the exception is the
dangerous part.

**The ingress gateway has a proxy.** It is the one pod in the cluster running
`istio-proxy`. A mesh-wide `PeerAuthentication` in the root namespace
(`platform-istio`) therefore applies to it. External clients arriving through the
NLB do not present client certificates, so STRICT at the gateway rejects inbound
traffic. The gateway is the platform's front door.

This is the failure mode to avoid: a policy that looks like a no-op everywhere,
because it very nearly is, and breaks the single component it does reach.

## Rollout sequence

Mirrors the policy lifecycle in the repository README — nothing goes straight to
its final state.

```
label a namespace for injection
  → restart workloads, confirm sidecars
  → PeerAuthentication PERMISSIVE
  → observe for one sprint
  → STRICT for that namespace
  → repeat
  → mesh-wide STRICT last, with the gateway exempted
```

1. **Injection first.** A namespace joins the mesh by carrying the revision
   label. This install is revisioned, so it is `istio.io/rev: 1-30-3`, not the
   unversioned `istio-injection: enabled`. Existing pods do not gain a sidecar
   until they restart.
2. **PERMISSIVE before STRICT.** PERMISSIVE accepts both mTLS and plaintext, so a
   half-migrated namespace keeps working. It also makes the eventual STRICT
   promotion a one-line, independently revertible change.
3. **One namespace at a time.** Blast radius is the point.
4. **Mesh-wide last**, and only with an explicit exemption for the ingress
   gateway's port.

## What is in this directory

| Path | Status |
|---|---|
| `mtls/` | PERMISSIVE PeerAuthentication per namespace. Inert until injection is enabled. |

## Not yet written

- `AuthorizationPolicy` default-deny per namespace — depends on namespaces being
  meshed first, otherwise it denies nothing.
- SPIFFE identity verification — needs at least two meshed workloads to verify
  between.
- Cross-namespace escalation test — the thing that proves the above works.

## The AWS half

`AuthorizationPolicy` and mTLS cover pod-to-pod traffic. They do **not** cover a
pod assuming an IAM role. That is the IRSA trust condition, which lives in
`infra-aws` rather than here, and the Kyverno guardrail in `kyverno/policies/`
which detects the Kubernetes side of the same problem.

Audited 14 August: all four IRSA trust policies in `infra-aws`
(`karpenter-controller`, `ebs-csi`, and the two GitHub OIDC roles) already carry
`StringEquals` conditions on both `:sub` and `:aud`. Nothing to retrofit today.
The open work is keeping it that way as roles are added.
