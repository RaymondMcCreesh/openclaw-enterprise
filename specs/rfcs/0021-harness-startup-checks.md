---
status: Proposed
implementation_status: Not implemented
author: RaymondMcCreesh
---

# Proposal: Standard Harness startup checks

- **ID:** RFC-0021
- **Created:** 2026-10-17
- **Last updated:** 2026-10-17
- **RFC PR:** [#2764](https://github.com/openclaw/openclaw-enterprise/pull/2764)
- **Implementation plan:** none; delivery is tracked [below](#delivery-and-verification)
- **Related:** [#96](https://github.com/openclaw/openclaw-enterprise/issues/96), [#1070](https://github.com/openclaw/openclaw-enterprise/pull/1070), [#2058](https://github.com/openclaw/openclaw-enterprise/pull/2058), [#2416](https://github.com/openclaw/openclaw-enterprise/issues/2416), [#2718](https://github.com/openclaw/openclaw-enterprise/pull/2718), [#2746](https://github.com/openclaw/openclaw-enterprise/pull/2746), [#2756](https://github.com/openclaw/openclaw-enterprise/pull/2756), [#2763](https://github.com/openclaw/openclaw-enterprise/pull/2763), [#2737](https://github.com/openclaw/openclaw-enterprise/pull/2737)

<a id="problem-and-decision"></a>

## Summary

Standardize three cached startup checks in exact-revision Agent diagnostics: `runtime-readiness`, `model-authentication`, and `model-connectivity`. The checks use the existing generic array and name the runtime container as `component` (`agent` or `gateway`), while `check` is a separate name with no slash. Kubernetes Compute reads latched evidence and never starts model or provider work for the request. Missing, replaced, or unreachable evidence is `unknown`, never `failed`.

## Motivation

At current `upstream/main` (`c2f3babc1`), runtime entrypoints latch only a failure in `runtimeFailure` and clear it on readiness; they do not record success (`apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:382-402`). The status endpoint exposes that held failure (`apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:420-428`). Exact-revision diagnostics return held failures and generic runtime checks, but do not expose the proposed cached Harness checks or deduplicate equivalent rows (`apps/controller/src/drivers/compute/kubernetes/index.ts:4064-4093`).

PR #1070 already persists classified model-probe failures in deployment status immediately. The worker maps held codes to deployment codes (`apps/controller/src/worker.ts:881-898`) and finalizes a held runtime failure without waiting for convergence timeout (`apps/controller/src/worker.ts:4657-4682`). This RFC builds on PR #2763's centralized runtime vocabulary: that contract currently defines `RUNTIME_COMPONENTS`, diagnostic codes, failure checks, and generic diagnostic checks (`packages/contracts/src/runtime-failure.mjs:10-56`). This RFC extends that file with the three check names and `READY` and `NOT_RUN`; it does not add a new component to `RUNTIME_COMPONENTS`. The proposed component remains the existing runtime container so dedicated mode preserves separate `agent` and `gateway` evidence.

The current generic contract already provides `component`, `check`, `state`, nullable `checkedAt`, and optional safe `code` (`packages/contracts/src/index.ts:1814-1828`). Diagnostics are a read path at the Compute boundary; they must not rerun startup work (`docs/reference/drivers/compute.md:152-164`).

<a id="scope"></a>

## Goals

- Return `component: "agent" | "gateway"` and one or more of `check: "runtime-readiness" | "model-authentication" | "model-connectivity"` in the existing generic `checks` array. Check names contain no `/`.
- Use `succeeded`, `failed`, or `unknown`. Every Harness entry has the latched probe `checkedAt` or `null` and a code from the closed vocabulary.
- Keep `checkedAt` distinct from diagnostics `observedAt`, current readiness, and serving health.
- Make missing, replaced, and unreachable runtimes safe and explicit, including OpenShell-provisioned runtimes.
- Preserve the held-failure ordering guarantee under the 32-check cap and suppress only a genuinely duplicated failure.

## Non-goals

This RFC does not persist Agent health or serving observations, change `Agent.status`, add gateway/xDS or Pod-restart checks, run live model or provider probes on diagnostics requests, expose provider output, add automatic repair, or define DNS-specific classification. DNS uses `MODEL_PROBE_FAILED` until a later contract decision adds a distinct code.

<a id="design"></a>

## Proposal

### Contract and mapping

The contract additions belong in `packages/contracts/src/runtime-failure.mjs`, alongside the lists introduced by #2763. Add `runtime-readiness`, `model-authentication`, and `model-connectivity` to `RUNTIME_DIAGNOSTIC_CHECKS`; add `READY` and `NOT_RUN` to `RUNTIME_DIAGNOSTIC_CODES`. Do not add `harness` to `RUNTIME_COMPONENTS`: the component identifies the container that owns the evidence, which preserves the two status authorities in dedicated mode. The resulting entries are:

| Component            | Check                  | Succeeded           | Failed                                                                    | Unknown                                        |
| -------------------- | ---------------------- | ------------------- | ------------------------------------------------------------------------- | ---------------------------------------------- |
| `agent` or `gateway` | `runtime-readiness`    | `succeeded / READY` | held startup code                                                         | `unknown / NOT_RUN` or `unknown / UNAVAILABLE` |
| `agent` or `gateway` | `model-authentication` | `succeeded / READY` | `AUTHENTICATION_FAILED` or `LOGIN_FAILED`                                 | `unknown / NOT_RUN` or `unknown / UNAVAILABLE` |
| `agent` or `gateway` | `model-connectivity`   | `succeeded / READY` | `MODEL_PROBE_TIMEOUT`, `MODEL_PROBE_CPU_STARVED`, or `MODEL_PROBE_FAILED` | `unknown / NOT_RUN` or `unknown / UNAVAILABLE` |

The full held-code mapping is: `model-probe/AUTHENTICATION_FAILED` maps to failed authentication; `model-probe/MODEL_PROBE_TIMEOUT`, `MODEL_PROBE_CPU_STARVED`, and `MODEL_PROBE_FAILED` map to failed connectivity; `login/AUTHENTICATION_FAILED` maps to failed authentication; `login/LOGIN_FAILED` maps to failed authentication with the provider-specific cause unresolved. `peer-bridge-record/UNAVAILABLE`, `plugin-approvers/INCOMPATIBLE_RESPONSE`, and `state-migration/UNAVAILABLE` map to failed runtime-readiness, with both model checks `unknown / NOT_RUN`. `workspace-setup/WORKSPACE_SETUP_REFUSED` is likewise failed runtime-readiness. These mappings follow the held code set and call sites (`packages/contracts/src/runtime-failure.mjs:13-34`, `apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:1190,2721-2723`, `apps/controller/src/drivers/compute/runtime-startup.ts:127-134`, `apps/controller/src/worker.ts:881-898`). A held failure is not a model failure merely because it uses `UNAVAILABLE` or `INCOMPATIBLE_RESPONSE`.

`UNAVAILABLE` has two state-qualified meanings and remains the smallest compatible vocabulary: `failed / UNAVAILABLE` means the runtime explicitly latched that startup failure; `unknown / UNAVAILABLE` means Compute could not reach or validate the status source. The state is the discriminator. A future distinct unreachable code remains a maintainer choice, not a reason to change this RFC's check shape.

Credential rejection makes authentication failed. Connectivity is succeeded only when the Driver can establish that the provider answered; otherwise it is unknown. Kubernetes Compute must use the provider-response marker from #2746 rather than infer provider ownership from a 401, because OpenShell may be the responder. #2756's response classification remains safe evidence only; its headers and body summaries do not become check fields or codes.

For runtime-auth Agents on Kubernetes Compute, readiness is succeeded after startup and both model checks are `unknown / NOT_RUN`: the operator-managed credential is not probed by this contract. Native OpenShell workers have no status port (`apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:4110-4125`, as documented by #2737); all three checks are therefore `unknown / UNAVAILABLE` and the deployment record is the only evidence. A provider-owned runtime must not be allowed to forge a succeeded row: until Compute has an authenticated, Driver-attested status path, its rows remain unknown. The Console labels succeeded rows as cached startup evidence, never as live serving health.

### Publication and compatibility

The startup code will add a separate `runtimeStartupChecks` field to the cached object returned by `/openclaw/runtime/status`; it will not overload `runtimeFailure`. The status endpoint is the correct seam because it is already the cached held-failure path (`apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:117-126,420-428`), while `/openclaw/runtime/diagnostics` currently performs the existing Slack channel read (`apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts:685-694,757-775`). Compute maps `runtimeStartupChecks` into the exact-revision generic array. Older controllers ignore the new field, and the #2718 held-failure transport continues to read only `runtimeFailure`, so a success latch can never be misread as a held failure.

The held gateway/model-probe row remains beside Harness checks unless it is the same failure. “Same” means the held row and a Harness row have the same safe code and the held check maps to that exact Harness check and failed state. An `UNAVAILABLE` held peer-bridge failure is not the same as an unreachable `unknown / UNAVAILABLE` row. Suppression happens only after this equality test. Held rows are emitted first, preserving their ordering guarantee before the final `slice(0, 32)` (`apps/controller/src/drivers/compute/kubernetes/index.ts:4084-4093`). If no status latch exists, older runtime images produce no Harness rows; the held row remains the fallback.

The OCC Sandbox Driver's lifecycle observation stays separate. It is the leading `agent`/`sandbox` check and describes Sandbox lifecycle, not model access (`packages/occ/src/index.ts:867-885`). Docker and SSH do not implement this Kubernetes status contract: Docker rejects Harness authentication (`apps/controller/src/drivers/compute/docker/index.ts:531-544`), and SSH supports embedded OpenClaw with operator-managed credentials only (`apps/controller/src/drivers/compute/ssh/index.ts:71-77,463-477`). They omit Harness rows rather than synthesize failures.

### Security and Console behavior

The #2746 and #2756 probe fields remain in sanitized log events only. They never become check fields or codes. Harness rows pass the masking and closed-vocabulary validation added by #2763; unknown component, check, and code values become `UNRECOGNIZED` rather than exposing provider text (`packages/occ/src/deployment-diagnostics.ts:24-48` and `packages/occ/src/runtime-failure-cause.ts:120-133`, as changed by #2763).

The Console will update the exact-revision detail copy at `apps/controller/src/console/agents/detail.mjs:913-945` and `:971-980`: gateway checks still say they cover Slack and do not test model credentials; the page will say Harness checks are latched startup evidence, not live serving state; and the failure footer will no longer say Harness checks cannot confirm `RUNTIME_AUTHENTICATION_FAILED`. Deployment-record failure remains authoritative for the recorded outcome. Diagnostics can add current cached evidence, but a successful Harness row cannot clear a deployment failure or override a later record. Older images show no Harness rows, so the existing held row and its fallback copy remain.

## Delivery and verification

Native API integration tests will exercise exact-revision authorization, healthy, credential rejection, timeout, provider failure, not-run, missing/replaced revisions, unreachable OpenShell status, code masking, held-row deduplication, and the assertion that a diagnostics request starts no model call, process, WebSocket, or provider probe. Runtime integration tests will verify the status latch and mixed-version behavior. Kubernetes conformance tests will cover embedded and dedicated container components. Console tests will render both Harness evidence and the existing Slack/Sandbox rows without claiming live health. The implementation must include the provider-response distinction for each Driver and redacted k3d proof for healthy and rejected credentials; other failure classes may remain test-only.

<a id="alternatives-and-open-decisions"></a>

## Rationale and alternatives

The existing generic array is preferred over a new field because it reuses the current authorization, masking, 32-check cap, and Console rendering paths. Naming the container rather than inventing `harness` preserves separate evidence for dedicated `agent` and `gateway` containers. Keeping the held row as a fallback preserves evidence from old images and non-Harness startup steps; the code-plus-mapping rule prevents duplicate display without hiding unrelated failures.

## Unresolved questions

Maintainers must confirm the container-component choice, `READY` and `NOT_RUN`, and whether the state-qualified `UNAVAILABLE` vocabulary is sufficient or should gain a distinct unreachable code. They must also confirm the per-Driver rule for calling connectivity succeeded after credential rejection and the Console treatment of provider-owned success. DNS-specific classification remains deferred; the fallback is `MODEL_PROBE_FAILED`.

## References

- [Runtime failure vocabulary from PR #2763](https://github.com/openclaw/openclaw-enterprise/pull/2763)
- [Compute Driver diagnostics](../../docs/reference/drivers/compute.md) (lines 152-164)
- [Runtime diagnostic contract from PR #2763](https://github.com/openclaw/openclaw-enterprise/blob/787bb9731d69cc79c9bbda6c7eb25f8dc5641396/packages/contracts/src/runtime-failure.mjs) (lines 10-56)
- [Kubernetes runtime status and diagnostics](../../apps/controller/src/drivers/compute/kubernetes/runtime-entrypoints.ts) (lines 382-428 and 685-775)
- [Kubernetes exact-revision diagnostics](../../apps/controller/src/drivers/compute/kubernetes/index.ts) (lines 4064-4093 and 9172-9237)
- [Runtime failure persistence](../../apps/controller/src/worker.ts) (lines 881-898 and 4657-4682)
- [Sandbox lifecycle diagnostics](../../packages/occ/src/index.ts) (lines 867-885)
