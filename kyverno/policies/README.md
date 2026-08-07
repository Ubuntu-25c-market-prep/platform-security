# Pod security baseline

`pod-security-baseline.yaml` is a Kyverno ClusterPolicy in Audit mode.
Target: platform-security issue #2.

## What it checks

- No privileged containers (`securityContext.privileged` must be false)
- No host namespace sharing (`hostPID`, `hostIPC`, `hostNetwork` must be false)

Both give a pod root-equivalent access to the node it runs on if allowed.

## Mode

Audit only — violations are reported via PolicyReport, nothing is blocked yet.
Per the promotion process in the repo README: watch for a sprint, fix what it
finds, then promote to Enforce in a separate PR.

## Verification

Applied to a local kind cluster. Deployed a test pod with `privileged: true`
(resource limits included to clear an unrelated pre-existing policy). Pod was
created successfully (Audit mode doesn't block), and `kubectl get polr -n default`
showed the violation: 2 checks passed, 1 failed — confirming the policy
correctly flagged the privileged container without blocking the pod.