# Required labels

`require-labels.yaml` is a Kyverno ClusterPolicy in Enforce mode.
Target: platform-security issue #3.

## What it checks

Every Pod must have two labels set and non-empty:
- `u25c.io/workstream`
- `u25c.io/owner`

These drive cost attribution and answer "who owns this" during an incident.

## Mode

Enforce — pods missing either label are rejected immediately, not just
reported. Unlike the pod-security-baseline policy, this went straight to
Enforce rather than Audit first, since missing labels don't break anything
at runtime — there's low risk in blocking right away, and cost tracking
depends on the labels being present everywhere, not "eventually."

## Verification

Applied to a local kind cluster. Deployed a test pod with no labels at all.
The pod was rejected outright by the admission webhook, with the exact
validation message defined in the policy ("Missing required label(s)...").
Confirms the policy blocks non-compliant pods rather than just reporting them.