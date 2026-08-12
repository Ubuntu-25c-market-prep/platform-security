# Restrict IRSA cross-namespace usage

`restrict-irsa-cross-namespace.yaml` is a Kyverno ClusterPolicy in Audit mode.
Target: platform-security issue #4.

## What it checks

Any ServiceAccount with an `eks.amazonaws.com/role-arn` annotation (i.e. one
wired up for IRSA) must have its own namespace name appear somewhere in that
role's ARN. ServiceAccounts without this annotation are skipped entirely.

## Why

The actual fix for cross-namespace IRSA abuse lives on the AWS side — an IAM
role's trust policy should be locked to a specific namespace:serviceaccount.
Kyverno can't inspect or enforce AWS trust policies directly. This is a
guardrail on the Kubernetes side: it makes a namespace/role mismatch obvious
and enforceable, rather than relying entirely on the AWS-side config being
set up correctly every time.

## Mode

Audit — this is a new convention, so we're watching what it catches on real
ServiceAccounts before blocking anything.

## Verification

Applied to a local kind cluster. Deployed a ServiceAccount in the `default`
namespace with a role ARN referencing `prod-payments-role` (no mention of
`default`). It was created successfully (Audit mode doesn't block), and
`kubectl get polr -n default` showed 1 failure for it — while Kubernetes'
own built-in `default` ServiceAccount (no IRSA annotation) showed as
correctly skipped rather than flagged, confirming the precondition works.