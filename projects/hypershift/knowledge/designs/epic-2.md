# Design: Fix Registry Override Not Applied When Extracting Release Metadata

**Epic:** #2 — Fix registry override not applied when extracting release metadata on non-OpenShift management clusters
**Upstream bug:** OCPBUGS-83564
**Date:** 2026-04-17

---

## Overview

On non-OpenShift management clusters (e.g., AKS in ARO-HCP), hosted cluster creation fails when using nightly OCP releases with `--registry-overrides` configured. The `RegistryMirrorProviderDecorator` applies registry overrides only to **component image tags** inside the returned `ImageStream`, but does **not** apply them to the **release image pullspec** before calling the delegate provider. This causes the system to query the original registry (`quay.io`) instead of the configured mirror (e.g., ACR), resulting in an `unauthorized` error for nightly releases that require authentication.

GA releases work because `quay.io/openshift-release-dev/ocp-release` is publicly accessible. Nightly releases (`ocp-release-nightly`) require authentication, exposing the bug.

---

## Phased Approach

### Verified vs. Hypothesized

Only one failure path is confirmed by the upstream bug report (OCPBUGS-83564): the `RegistryMirrorProviderDecorator.Lookup()` call in the HO's `GetControlPlaneOperatorImage` path. The extended code audit identified additional code paths that appear to have the same class of bug, but these have **not been reproduced** in a live environment. Some components (HCCO, karpenter, CPO) use a workaround — merging `--registry-overrides` into the ICSP/IDMS map — whose real-world sufficiency is unverified.

Before expanding scope, we must reproduce each hypothesized failure on an actual AKS management cluster.

### Phase 0: Reproduction on AKS

**Goal:** Confirm which code paths actually fail on a non-OpenShift management cluster with `--registry-overrides` + nightly release.

**Prerequisites:**
1. Provision an AKS management cluster (no ICSP/IDMS capability)
2. Set up an ACR mirror containing a nightly OCP release (`ocp-release-nightly`)
3. Install HyperShift operator with `--registry-overrides=quay.io/openshift-release-dev/ocp-release-nightly=<acr-mirror>/openshift-release-dev/ocp-release-nightly`
4. Configure a HostedCluster with matching `imageContentSources`

**Reproduction matrix:**

| # | Code Path | Component | What to observe | Expected failure (hypothesis) |
|---|-----------|-----------|----------------|-------------------------------|
| R1 | `GetControlPlaneOperatorImage` → `Lookup()` | HO HC Reconciler | HC condition `ValidReleaseImage` | `unauthorized` pulling from `quay.io` (confirmed by OCPBUGS-83564) |
| R2 | `DetermineHostedClusterPayloadArch` → `GetManifest()` | HO HC Reconciler | HC condition `ValidReleaseImage=False`, reconciler halts | `unauthorized` — metadata provider has no `--registry-overrides` support |
| R3 | `GetDigest` for Progressing condition | HO HC Reconciler | HC condition `Progressing=Blocked` permanently | `unauthorized` — metadata provider early-return path |
| R4 | Karpenter controller `Lookup()` | Karpenter Operator | Karpenter reconciler error log | `unauthorized` — bare `RegistryClientProvider`, zero decorators |
| R5 | Karpenter ignition controller `Lookup()` | Karpenter Operator | Ignition reconciler error log | Broken natively — `RegistryOverrides: nil`, outer decorator workaround may mask |
| R6 | HCCO release provider `Lookup()` | HCCO | HCCO reconciler error log | Broken natively — `RegistryOverrides: nil` (same class as R5), outer decorator workaround may mask; HCCO is core production, always deployed |

**Method:** For R1-R3, create a HostedCluster and observe the HO reconciler logs and HC conditions. R2 likely fires before R1 (arch detection runs first in the reconcile loop). For R4-R5, enable karpenter on the cluster. For R6, observe HCCO logs in the hosted control plane namespace.

**Outcome:** Each row gets a status: **confirmed** (failure reproduced), **not affected** (workaround sufficient), or **blocked** (can't reach this path because an earlier path fails first). Confirmed paths get fixes; not-affected paths get documented; blocked paths get retested after earlier fixes land.

### Phase 1: Confirmed Fix

Fix 1 (`RegistryMirrorProviderDecorator.Lookup()`) is confirmed by OCPBUGS-83564 and is implemented (commit `c804b0c17`). This fix lands regardless of reproduction results.

### Phase 2: Validated Fixes

Fixes 2-6 are conditional on Phase 0 reproduction results. Each fix is applied only if its corresponding reproduction row is confirmed as a real failure. The design below specifies each fix so they are ready to implement once validated.

---

## Architecture

### Current Provider Chain

The release provider is a decorator chain constructed in `NewCommonRegistryProvider` (`support/globalconfig/imagecontentsource.go:226-261`):

```
ProviderWithOpenShiftImageRegistryOverridesDecorator     (ICSP/IDMS mirrors)
  -> RegistryMirrorProviderDecorator                     (--registry-overrides flag)
       -> CachedProvider
            -> RegistryClientProvider                    (actual registry pull)
```

### Current Call Flow (Broken)

```
GetControlPlaneOperatorImage()                           [support/util/util.go:667]
  -> releaseProvider.Lookup(image="quay.io/.../ocp-release-nightly@sha256:...")
    -> ProviderWithOpenShiftImageRegistryOverridesDecorator.Lookup()
       [registry_image_content_policies.go:23]
       OpenShiftImageRegistryOverrides is nil on non-OpenShift -> falls through
      -> RegistryMirrorProviderDecorator.Lookup()
         [registry_mirror_provider.go:26]
         Calls Delegate.Lookup(image) with ORIGINAL image   <-- BUG
         Only post-processes component tags in returned ImageStream
        -> CachedProvider.Lookup()
          -> RegistryClientProvider.Lookup()
             [registryclient_provider.go:22]
             Calls ExtractImageFiles(image="quay.io/...") -> UNAUTHORIZED
```

### Fixed Call Flow

```
GetControlPlaneOperatorImage()
  -> releaseProvider.Lookup(image="quay.io/.../ocp-release-nightly@sha256:...")
    -> ProviderWithOpenShiftImageRegistryOverridesDecorator.Lookup()
       [no-op on non-OpenShift]
      -> RegistryMirrorProviderDecorator.Lookup()
         Applies RegistryOverrides to image BEFORE delegating  <-- FIX
         Calls Delegate.Lookup(lookupImage="acr.io/...@sha256:...")
        -> CachedProvider.Lookup()
          -> RegistryClientProvider.Lookup()
             Calls ExtractImageFiles(image="acr.io/...") -> SUCCESS
```

### Systemic Gap Analysis (Hypothesized — Pending Phase 0 Reproduction)

Code audit identified additional code paths that appear to have the same class of bug as the confirmed `Lookup()` fix. **None of these have been reproduced in a live environment.** Some components use a workaround (merging `--registry-overrides` into the ICSP/IDMS map) whose sufficiency in the AKS + nightly scenario is unverified. Each hypothesis maps to a reproduction row in Phase 0.

#### Why ICSP/IDMS Is Not Affected

The two override mechanisms are wired differently:

- **ICSP/IDMS** was built as a **two-pronged mechanism**: (1) `ProviderWithOpenShiftImageRegistryOverridesDecorator.Lookup()` applies overrides to the image *before* delegating in the release info path, and (2) `RegistryClientImageMetadataProvider` has its own `OpenShiftImageRegistryOverrides` field with every method calling `SeekOverride()` before registry access.

- **`--registry-overrides`** was only built into the release info path via `RegistryMirrorProviderDecorator`, and even there it was applied *after* the delegate call (to tags only). It was **never added** to `RegistryClientImageMetadataProvider`. Three components (CPO, HCCO, karpenter-operator) noticed this gap and worked around it by merging `--registry-overrides` values into the ICSP/IDMS-style `map[string][]string` — effectively abandoning the native mechanism and piggybacking on the ICSP/IDMS path. But this workaround only covers scenarios where ICSP/IDMS infrastructure is available.

#### Hypothesis 1: `DetermineHostedClusterPayloadArch` — Potentially FATAL (→ R2)

**File:** `hypershift-operator/controllers/hostedcluster/hostedcluster_controller.go:1213`

`DetermineHostedClusterPayloadArch` calls `registryclient.IsMultiArchManifestList(ctx, hc.Spec.Release.Image, ...)` which passes the raw image through `RegistryClientImageMetadataProvider.GetManifest()`. This calls `SeekOverride()` for ICSP/IDMS but has no mechanism for `--registry-overrides`. On failure, the reconciler sets `ValidReleaseImage=False` and **returns the error** (line 1224), halting reconciliation entirely. **If confirmed, this blocks cluster creation before the `GetControlPlaneOperatorImage` fix is even reached.**

- **ICSP/IDMS:** Protected — `GetManifest()` calls `SeekOverride()` at `imagemetadata.go:262`
- **`--registry-overrides`:** No support in `RegistryClientImageMetadataProvider`
- **Reproduction:** R2 in Phase 0

#### Hypothesis 2: `GetDigest` for Progressing Condition — Potentially Operationally Fatal (→ R3)

**File:** `hypershift-operator/controllers/hostedcluster/hostedcluster_controller.go:1243`

`registryClientImageMetadataProvider.GetDigest(ctx, hcluster.Spec.Release.Image, pullSecretBytes)` passes the raw image. When this fails, the Progressing condition is set to `Reason: Blocked` permanently — monitoring alerts fire, upgrade orchestration stalls.

- **ICSP/IDMS:** Protected — `GetDigest()` calls `SeekOverride()` at `imagemetadata.go:216`
- **`--registry-overrides`:** No support — same root cause as Hypothesis 1
- **Reproduction:** R3 in Phase 0

#### Hypothesis 3: Karpenter Controller — Potentially FATAL (→ R4)

**File:** `karpenter-operator/main.go:130`

`ReleaseProvider: &releaseinfo.RegistryClientProvider{}` — a bare provider with **zero decorators**. Neither ICSP/IDMS nor `--registry-overrides` are applied. The `--registry-overrides` flag IS accepted (line 67) but never wired to this provider.

- **ICSP/IDMS:** Also no support — no decorator chain at all
- **`--registry-overrides`:** No support — no decorator chain at all
- **Reproduction:** R4 in Phase 0

#### Hypothesis 4: Karpenter Ignition Controller — Unclear (→ R5)

**File:** `karpenter-operator/main.go:161-172`

The decorator chain has `RegistryOverrides: nil` on the inner `RegistryMirrorProviderDecorator`. The `--registry-overrides` flag values are merged into `imageRegistryOverrides` (the ICSP/IDMS-style `map[string][]string`, lines 154-159) instead of being wired to the inner decorator. The outer decorator has the merged values and may handle the override successfully — this workaround's sufficiency on AKS is unverified.

- **ICSP/IDMS:** Outer decorator has merged flag values
- **`--registry-overrides`:** Inner decorator receives `nil`, flag values rerouted through ICSP/IDMS semantics
- **Reproduction:** R5 in Phase 0 — may work via the outer decorator workaround

#### Hypothesis 5: `RegistryClientImageMetadataProvider` — Systemic Root Cause of H1/H2

**File:** `support/util/imagemetadata.go:115-119`

The struct has `OpenShiftImageRegistryOverrides map[string][]string` but **no field for `--registry-overrides`** (`map[string]string`). Every method (`ImageMetadata`, `GetOverride`, `GetDigest`, `GetManifest`, `GetMetadata`) calls `SeekOverride()` which only processes ICSP/IDMS entries. If H1/H2 are confirmed, this is the root cause — any caller that resolves images through the metadata provider bypasses `--registry-overrides` entirely.

Additionally, `GetDigest` (line 205) and `GetMetadata` (line 288) have **early-return paths** gated on `len(r.OpenShiftImageRegistryOverrides) == 0`. When ICSP/IDMS is empty (non-OpenShift clusters), these methods return immediately — bypassing `SeekOverride()` entirely. Any `--registry-overrides` support added to this struct must be applied **before** these early-return conditions, and the conditions themselves must be updated to also check the new `RegistryOverrides` field.

- **ICSP/IDMS:** Protected by design — every method calls `SeekOverride()`
- **`--registry-overrides`:** Not supported at all
- **Reproduction:** Validated indirectly via R2/R3 in Phase 0

#### Hypothesis 6: HCCO Release Provider — Broken Natively (→ R6)

**File:** `control-plane-operator/hostedclusterconfigoperator/cmd.go:272`

The HCCO release provider chain has `RegistryOverrides: nil` on its `RegistryMirrorProviderDecorator` (line 272). HCCO accepts `--registry-overrides` (line 156) and stores the values in `o.registryOverrides`, but instead of wiring them to the inner decorator, merges them into `imageRegistryOverrides` (lines 252-261) — the ICSP/IDMS-style `map[string][]string`. The inner `RegistryMirrorProviderDecorator` receives `nil` for `RegistryOverrides`, so its native string-replacement path is disabled.

This is the **exact same pattern** as Finding 4 (karpenter ignition controller): the `--registry-overrides` flag value exists on the struct (`o.registryOverrides`, line 123) but is rerouted through ICSP/IDMS semantics instead of being wired to the mechanism designed for it. On non-OpenShift management clusters where ICSP/IDMS infrastructure is unavailable, the workaround through the outer decorator may function (since the merged values are present), but the native mechanism is definitively broken.

HCCO is a **core production component** — it runs inside every hosted control plane and reconciles resources in the hosted cluster. Unlike karpenter (which is opt-in), HCCO is always deployed. Any failure in HCCO's release provider affects every hosted cluster on a non-OpenShift management cluster using `--registry-overrides` with nightly releases.

- **ICSP/IDMS:** Outer decorator has merged flag values — may function as workaround
- **`--registry-overrides` native path:** Broken — inner decorator receives `nil`, flag values rerouted through ICSP/IDMS semantics
- **Reproduction:** R6 in Phase 0
- **Risk if unfixed:** Even if the outer decorator workaround functions today, it changes `--registry-overrides` semantics (adds 15s mirror probing timeout, availability caching, fallback ordering). Any future refactor that splits the mechanisms or removes the merge workaround would silently break HCCO. The fix is trivial (one-line wire) and eliminates this fragility.

#### Impact Matrix (Hypothesized — Pending Phase 0)

| Code Path | Component | `--registry-overrides` | ICSP/IDMS | Hypothesized Severity | Repro Row |
|-----------|-----------|----------------------|-----------|-----------------------|-----------|
| `Lookup()` (release info) | HO HC Reconciler | **Fixed** (commit c804b0c17) | Works | **Confirmed** — resolved | — |
| `DetermineHostedClusterPayloadArch` | HO HC Reconciler | No metadata support | Works | Potentially FATAL | R2 |
| `GetDigest` for Progressing | HO HC Reconciler | No metadata support | Works | Potentially operationally fatal | R3 |
| Karpenter controller `Lookup` | Karpenter Operator | Bare provider, no decorators | No decorators | Potentially FATAL | R4 |
| Karpenter ignition `Lookup` | Karpenter Operator | `nil` (workaround in outer) | Works (via workaround) | Broken natively — needs repro | R5 |
| HCCO `Lookup` (release info) | HCCO | `nil` (workaround in outer) | Works (via workaround) | Broken natively — core production component, always deployed | R6 |
| NodePool reconciler (all paths) | HO NodePool Reconciler | Works | Works | OK | — |
| CPO `cpReleaseProvider` | CPO | Works | Works | OK | — |
| CPO `userReleaseProvider` | CPO | `nil` (intentional) | Works | OK (by design) | — |
| Ignition server | Ignition Server | Works | Works | OK | — |

---

## Components and Interfaces

### Confirmed Fix (Phase 1)

#### Fix 1: `RegistryMirrorProviderDecorator.Lookup()` — `support/releaseinfo/registry_mirror_provider.go:26-46`

Apply `RegistryOverrides` string replacement to the `image` parameter **before** calling `p.Delegate.Lookup()`, in addition to the existing post-processing of component image tags.

**Status:** Implemented in commit `c804b0c17`. Lands regardless of Phase 0 results.

### Conditional Fixes (Phase 2 — Pending Phase 0 Reproduction)

The following fixes are applied only if their corresponding reproduction rows confirm a real failure.

#### Fix 2: `RegistryClientImageMetadataProvider` — `support/util/imagemetadata.go:115-119` (if R2/R3 confirmed)

Add a `RegistryOverrides map[string]string` field. Apply simple string replacement to the image reference at the **top of each method**, before any early-return logic or `reference.Parse()` calls. This is critical because `GetDigest` (line 205) and `GetMetadata` (line 288) have early-return paths gated on `len(r.OpenShiftImageRegistryOverrides) == 0` — on non-OpenShift clusters, these methods bypass `SeekOverride()` entirely.

**Implementation detail for `GetDigest` (lines 188-249):**

The current `GetDigest` control flow on non-OpenShift clusters (empty `OpenShiftImageRegistryOverrides`) is:

```
GetDigest(imageRef) →
  Parse(imageRef)                                        // line 198
  if len(OpenShiftImageRegistryOverrides) == 0 {         // line 205 — TRUE on non-OpenShift
    if cache hit for imageRef → return cached digest      // lines 207-209
    ref = &parsedImageRef                                 // line 212
  }                                                       // falls through to line 216
  ref = SeekOverride(OpenShiftImageRegistryOverrides, parsedImageRef, ...)  // line 216 — NO-OP (empty map)
  // continues with unoverridden ref...
```

The early-return at line 205 has two effects when the condition is true:
1. **Cache-hit path (line 207-209):** Returns immediately with the **unoverridden** `imageRef` digest. If `--registry-overrides` string replacement has not yet been applied, the cache lookup uses the wrong key (original registry, not mirror) and the returned reference points to the wrong registry.
2. **Cache-miss path (line 212):** Sets `ref = &parsedImageRef` and falls through to line 216, where `SeekOverride()` is a no-op (empty ICSP/IDMS map). Execution continues using the **unoverridden** parsed reference, causing `getRepoSetup` at line 225 to connect to the original registry.

**Required changes:**

1. Apply `--registry-overrides` string replacement to `imageRef` at the **top of the method** (before `reference.Parse(imageRef)` at line 198). This ensures every subsequent use of `imageRef` and `parsedImageRef` operates on the overridden value.

2. Update the early-return condition at line 205 from:
```go
if len(r.OpenShiftImageRegistryOverrides) == 0 {
```
to:
```go
if len(r.OpenShiftImageRegistryOverrides) == 0 && len(r.RegistryOverrides) == 0 {
```
This ensures the early-return block is only entered when **neither** override mechanism is configured. When `RegistryOverrides` is non-empty (even if ICSP/IDMS is empty), execution must fall through to `SeekOverride()` — which is a no-op for ICSP/IDMS but ensures the method follows the normal code path where the already-overridden `imageRef` is processed correctly.

Without both changes, `GetDigest` would have **no effect** from Fix 2 for the target scenario: on non-OpenShift clusters, `OpenShiftImageRegistryOverrides` is empty, so the early-return fires, and even if `imageRef` was rewritten at the top, the cache-hit path returns before `SeekOverride()` is reached, while the cache-miss path sets `ref` from `parsedImageRef` (which IS correct if the replacement was applied before `Parse`). The condition update is still necessary because without it, the cache-hit path on line 207-209 checks `digestCache.Get(imageRef)` — which now contains the **overridden** imageRef — but then returns at line 209 with `parsedImageRef.ID` set from the cached digest. If the condition is not updated and `RegistryOverrides` is non-empty, the early-return block bypasses the `SeekOverride` + `getRepoSetup` path that would be reached on cache miss, creating inconsistent behavior between cache-hit and cache-miss.

**Implementation detail for `GetMetadata` (lines 281-302):**

The current `GetMetadata` early-return at line 288 is **unconditional** — it does not check the cache:

```
GetMetadata(imageRef) →
  if len(OpenShiftImageRegistryOverrides) == 0 {         // line 288 — TRUE on non-OpenShift
    return getMetadata(ctx, imageRef, pullSecret)          // line 289 — returns immediately
  }
  // lines 291+ are NEVER reached on non-OpenShift
```

This is the **most critical early-return** because it is unconditional: when the condition is true, `SeekOverride()` at line 298 is NEVER executed regardless of cache state. The `imageRef` passed to `getMetadata()` at line 289 is the raw, unoverridden value.

**Required changes (same pattern as `GetDigest`):**

1. Apply `--registry-overrides` string replacement to `imageRef` at the **top of the method**, before the early-return at line 288.

2. Update the condition from:
```go
if len(r.OpenShiftImageRegistryOverrides) == 0 {
```
to:
```go
if len(r.OpenShiftImageRegistryOverrides) == 0 && len(r.RegistryOverrides) == 0 {
```

With both changes: when `RegistryOverrides` is non-empty and ICSP/IDMS is empty, the condition is false, execution falls through to `SeekOverride()` (a no-op for ICSP/IDMS), and `getMetadata()` at line 300 receives the overridden `composedRef`. Alternatively, if only the string replacement is applied at the top, the early-return path at line 289 would pass the **already-overridden** `imageRef` to `getMetadata()`, which also produces the correct result. Both changes together provide defense-in-depth.

**Implementation detail for `ImageMetadata`, `GetOverride`, `GetManifest`:** These methods call `SeekOverride()` unconditionally (no early-return path) but `--registry-overrides` replacement must still be applied to `imageRef` before `reference.Parse()`, so the parsed reference passed to `SeekOverride()` already contains the overridden registry.

**Override ordering note:** In the release info decorator chain, ICSP/IDMS (outer decorator) runs first, then `--registry-overrides` (inner decorator) runs second. In the metadata provider, `--registry-overrides` string replacement is applied first (at the top of each method), then `SeekOverride()` for ICSP/IDMS runs second. This means the precedence between the two mechanisms is **reversed** between the release info and metadata paths. On non-OpenShift clusters (the target scenario for this epic) this is irrelevant because ICSP/IDMS is empty, so `SeekOverride()` is a no-op and only `--registry-overrides` has any effect. On dual-mechanism deployments where the same source registry is matched by both mechanisms, the reversed precedence could produce different override results depending on the code path. This is documented as a known limitation — see the Risks and Known Limitations section.

This resolves Hypotheses 1, 2, and 5 if R2/R3 are confirmed.

#### Fix 3: Karpenter controller wiring — `karpenter-operator/main.go:127-131` (if R4 confirmed)

Replace the bare `&releaseinfo.RegistryClientProvider{}` with the full decorator chain, matching the pattern already used for the karpenter ignition controller at lines 161-172. This resolves Hypothesis 3.

#### Fix 4: Karpenter ignition controller wiring — `karpenter-operator/main.go:169` (if R5 confirmed)

Wire `registryOverrides` to `RegistryMirrorProviderDecorator.RegistryOverrides` instead of (or in addition to) merging into the ICSP/IDMS map. This resolves Hypothesis 4. **Note:** R5 may show the outer decorator workaround is sufficient, in which case this fix is unnecessary.

#### Fix 5: `RegistryClientImageMetadataProvider` wiring — all instantiation sites (if R2/R3 confirmed)

Pass `registryOverrides` to the new `RegistryOverrides` field on `RegistryClientImageMetadataProvider` at each instantiation site:

| Site | File | Current | Fix |
|------|------|---------|-----|
| HO | `support/globalconfig/imagecontentsource.go:251` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| Karpenter | `karpenter-operator/main.go:174` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| CPO | `control-plane-operator/main.go:479` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| HCCO | `hostedclusterconfigoperator/cmd.go:277` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| Ignition server | `ignition-server/cmd/start.go:155` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |

#### Fix 6: HCCO release provider wiring — `control-plane-operator/hostedclusterconfigoperator/cmd.go:272`

Wire `o.registryOverrides` to `RegistryMirrorProviderDecorator.RegistryOverrides` at line 272 (currently `nil`). HCCO accepts `--registry-overrides` (line 156), stores it as `o.registryOverrides` (line 123), but currently only merges the values into the ICSP/IDMS-style `imageRegistryOverrides` map (lines 252-261) and passes `nil` to the inner decorator's `RegistryOverrides` field.

**Justification:** HCCO is a core production component deployed in every hosted control plane. Unlike karpenter (opt-in), HCCO always runs. The bug is the same class as Finding 4 (karpenter ignition) — `--registry-overrides` values exist on the struct but are routed through ICSP/IDMS semantics instead of the native mechanism. The fix is a one-line change:

```go
// Before (line 272):
RegistryOverrides: nil,

// After:
RegistryOverrides: o.registryOverrides,
```

Even if the outer decorator workaround masks this bug in current deployments, wiring the native path is necessary to: (a) avoid silently changing `--registry-overrides` semantics by routing through ICSP/IDMS mirror probing, (b) prevent breakage if the merge workaround is ever refactored out, and (c) maintain consistency with the fix applied to karpenter's equivalent pattern (Fix 4). This resolves Hypothesis 6.

### Interface Contracts (Unchanged)

| Interface | File | Change |
|-----------|------|--------|
| `Provider` | `support/releaseinfo/releaseinfo.go` | No change |
| `ProviderWithRegistryOverrides` | `support/releaseinfo/releaseinfo.go` | No change |
| `ProviderWithOpenShiftImageRegistryOverrides` | `support/releaseinfo/releaseinfo.go` | No change |

### Affected Callers (No Changes Required)

All callers using the full decorator chain benefit from the fix transparently:

| Caller | File | Provider Type |
|--------|------|---------------|
| HostedClusterReconciler | `hostedcluster_controller.go:672` | Full decorator chain |
| NodePoolReconciler | `nodepool_controller.go:980` | Full decorator chain |
| HAProxy reconciler | `haproxy.go:66` | Full decorator chain |
| HostedClusterSizing reconciler | `hostedclustersizing_controller.go:76` | Full decorator chain |

### Direct `RegistryClientProvider` Usages

The following callers instantiate `RegistryClientProvider` directly, bypassing the decorator chain entirely.

| Caller | File | Risk | Scope Decision |
|--------|------|------|----------------|
| karpenter-operator Reconciler | `karpenter-operator/main.go:130` | **Production** — same bug class | **Conditional** — Fix 3 if R4 confirmed |
| `hypershift create nodepool` CLI | `cmd/nodepool/core/create.go:82` | **Production** — CLI validation path | Out of scope — follow-up issue |
| NodePool upgrade E2E test | `test/e2e/nodepool_upgrade_test.go:166` | Low — test infra | Out of scope |
| Karpenter E2E test | `test/e2e/karpenter_test.go:481` | Low — test infra | Out of scope |
| Karpenter CP upgrade E2E test | `test/e2e/karpenter_control_plane_upgrade_test.go:56` | Low — test infra | Out of scope |
| E2E version util | `test/e2e/util/version.go:55` | Low — test infra | Out of scope |
| E2E util | `test/e2e/util/util.go:3936` | Low — test infra | Out of scope |

**Note:** The karpenter-operator file has *two* `RegistryClientProvider` usages — line 130 (direct, no decorators) and line 165 (properly wrapped in the full chain). Only the direct usage at line 130 is affected.

### Reference: Existing Override Pattern

`ProviderWithOpenShiftImageRegistryOverridesDecorator.Lookup()` (`registry_image_content_policies.go:23-54`) already applies a similar pattern for ICSP/IDMS overrides — it replaces the `image` string before calling `p.Delegate.Lookup()`. The fix aligns `RegistryMirrorProviderDecorator` with this established pattern.

Similarly, `RegistryClientImageMetadataProvider.ImageMetadata()` (`support/util/imagemetadata.go:131-168`) correctly uses `SeekOverride()` to apply ICSP/IDMS overrides before accessing the registry. The release info path should behave consistently.

---

## Data Models

No data model changes are required. The `RegistryOverrides` map (`map[string]string`) already contains the correct source-to-destination mappings. The fix only changes **when** these mappings are applied during the `Lookup` call.

---

## Error Handling

### Current Behavior (Broken)

When the override is not applied, `RegistryClientProvider.Lookup()` fails with:
```
failed to extract release metadata: failed to obtain root manifest for
quay.io/openshift-release-dev/ocp-release-nightly@sha256:...
unauthorized: access to the requested resource is not authorized
```

This error propagates up through `GetControlPlaneOperatorImage()` as:
```
failed to get controlPlaneOperatorImage: failed to extract release metadata: ...
```

### Fixed Behavior

With the override applied, the registry client connects to the mirror registry (e.g., ACR) which has valid credentials configured, and the lookup succeeds. If the mirror also fails, the error message will now correctly reference the mirror URL rather than the original registry, making debugging easier.

### Edge Cases

1. **No matching override:** If no `RegistryOverrides` entry matches the image, the original image is passed unmodified — preserving current behavior for GA releases and configurations without overrides.
2. **Multiple matching overrides:** `strings.Replace` with count=1 ensures only the first match is replaced per override entry. The `for` loop iterates over a Go `map[string]string`, so iteration order is non-deterministic. If two entries both match (e.g., `quay.io` and `quay.io/openshift-release-dev`), the result may vary between runs. This is a **known limitation**, pre-existing in the tag post-processing path, and is inherited unchanged by the fix. See the Risks and Known Limitations section.
3. **Cache key consistency:** The `CachedProvider` caches by the image string it receives. After the fix, the cache key will be the **overridden** image URL. This is correct — the same mirrored image should return the same cached result. Different mirror targets will produce different cache entries, which is also correct.

---

## Acceptance Criteria

### AC1: Registry override applied to release image lookup

**Given** a HyperShift operator configured with `--registry-overrides=quay.io/openshift-release-dev/ocp-release-nightly=mirror.example.com/openshift-release-dev/ocp-release-nightly`
**When** a HostedCluster is created referencing a nightly release image `quay.io/openshift-release-dev/ocp-release-nightly@sha256:abc123`
**Then** the release metadata extraction call uses `mirror.example.com/openshift-release-dev/ocp-release-nightly@sha256:abc123` instead of the original `quay.io` reference

### AC2: Nightly releases work on non-OpenShift management clusters

**Given** a non-OpenShift management cluster (e.g., AKS) with no ICSP/IDMS capability
**And** `--registry-overrides` configured to map `quay.io` nightly images to an accessible mirror
**When** a HostedCluster is created with a nightly OCP release
**Then** the `controlPlaneOperatorImage` is successfully resolved without authorization errors

### AC3: GA release behavior unchanged (no regression)

**Given** a HyperShift operator with `--registry-overrides` configured for nightly images only
**When** a HostedCluster is created with a GA release image (`quay.io/openshift-release-dev/ocp-release`)
**Then** the release metadata extraction uses the original GA image reference (no override match) and succeeds as before

### AC4: Component image tags still overridden

**Given** a HyperShift operator with `--registry-overrides` configured
**When** release metadata is successfully extracted
**Then** the component image tags in the returned `ImageStream` still have registry overrides applied (existing post-processing behavior preserved)

### AC5: Unit test coverage

**Given** `RegistryMirrorProviderDecorator` currently has **zero unit test coverage** (the only test that instantiates it, `TestProviderWithOpenShiftImageRegistryOverridesDecorator_Lookup`, passes empty `RegistryOverrides` and never exercises the override logic)
**When** the fix is implemented
**Then** new unit tests MUST land **with** the fix (not as follow-up), covering:
- Override is applied to the image passed to the delegate
- Override is NOT applied when no matching entry exists
- Component tag post-processing still works
- Multiple overrides are applied correctly
- Delegate receives the overridden image (not the original)

### AC6: ICSP/IDMS decorator interaction preserved

**Given** an OpenShift management cluster with ICSP/IDMS configured
**And** `--registry-overrides` also configured
**When** the decorator chain processes a release image lookup
**Then** ICSP/IDMS overrides are attempted first (outer decorator), and `--registry-overrides` are applied to the image before the inner delegate call — both layers work together without conflict

### AC7: INFO-level logging on override application

**Given** a `RegistryMirrorProviderDecorator` configured with `--registry-overrides`
**When** an override matches and the image reference is rewritten before delegation
**Then** an INFO-level log line is emitted showing the original image and the overridden image, matching the logging pattern established by `ProviderWithOpenShiftImageRegistryOverridesDecorator` (which logs at `registry_image_content_policies.go:42`)

### Conditional Acceptance Criteria (Phase 2 — Applied Only If Corresponding Reproduction Confirms Failure)

### AC8: Image metadata path supports `--registry-overrides` (if R2/R3 confirmed)

**Given** `RegistryClientImageMetadataProvider` configured with `--registry-overrides` mapping `quay.io/openshift-release-dev/ocp-release-nightly` to an accessible mirror
**When** `ImageMetadata()`, `GetDigest()`, or `GetManifest()` is called with a nightly release image
**Then** the override is applied to the image reference before registry access, and the call succeeds against the mirror

### AC9: `DetermineHostedClusterPayloadArch` succeeds on non-OpenShift clusters (if R2 confirmed)

**Given** a non-OpenShift management cluster with `--registry-overrides` configured for nightly images
**When** a HostedCluster is created with a nightly OCP release
**Then** `DetermineHostedClusterPayloadArch` resolves the payload architecture successfully without `ValidReleaseImage=False`

### AC10: Karpenter controller uses decorator chain (if R4 confirmed)

**Given** the karpenter-operator configured with `--registry-overrides`
**When** the karpenter controller reconciles a HostedControlPlane with a nightly release
**Then** the `ReleaseProvider.Lookup()` call applies registry overrides (not bare `RegistryClientProvider`)

### AC11: Karpenter ignition controller wires `--registry-overrides` natively (if R5 confirmed)

**Given** the karpenter-operator configured with `--registry-overrides`
**When** the karpenter ignition controller processes a release image lookup
**Then** `RegistryMirrorProviderDecorator.RegistryOverrides` is populated (not `nil`) and overrides are applied via native string replacement

### AC12: HCCO release provider wires `--registry-overrides` natively

**Given** HCCO configured with `--registry-overrides`
**When** HCCO's release provider processes a release image lookup
**Then** `RegistryMirrorProviderDecorator.RegistryOverrides` is populated with `o.registryOverrides` (not `nil`) and overrides are applied via native string replacement, independent of ICSP/IDMS availability

**Note:** HCCO is a core production component deployed in every hosted control plane. This AC is not conditional on R6 reproduction — the native mechanism is definitively broken (`RegistryOverrides: nil` at `cmd.go:272`), and the fix is a one-line wire that eliminates fragility regardless of whether the outer decorator workaround masks the failure in current deployments.

---

## Impact on Existing System

### Blast Radius

**Phase 1 (confirmed):**

| Change | Files | Nature |
|--------|-------|--------|
| Fix 1: `RegistryMirrorProviderDecorator.Lookup()` | 1 file | Behavioral — apply override before delegation |

**Phase 2 (conditional on Phase 0 reproduction):**

| Change | Files | Nature | Condition |
|--------|-------|--------|-----------|
| Fix 2: `RegistryClientImageMetadataProvider` | 1 file | Structural — add `RegistryOverrides` field, apply in 5 methods | R2/R3 confirmed |
| Fix 3: Karpenter controller wiring | 1 file | Wiring — replace bare provider with decorator chain | R4 confirmed |
| Fix 4: Karpenter ignition wiring | 1 file | Wiring — populate `RegistryOverrides` field | R5 confirmed |
| Fix 5: Metadata provider wiring | 5 files | Wiring — pass `registryOverrides` to metadata provider | R2/R3 confirmed |
| Fix 6: HCCO release provider wiring | 1 file | Wiring — populate `RegistryOverrides` field | R6 confirmed |

No interface changes, no new dependencies, no configuration changes. If Fix 2 is applied, the `RegistryClientImageMetadataProvider` struct gains one new field — existing callers that don't set it get the zero value (`nil`), preserving current behavior.

### Behavioral Changes

**Phase 1 (confirmed):**

| Scenario | Before | After |
|----------|--------|-------|
| Nightly release + `--registry-overrides` + non-OpenShift cluster | **FAILS** — pulls from original registry | **WORKS** — pulls from mirror |
| GA release + `--registry-overrides` (no matching entry) | Works (public access) | Works (unchanged — no override match) |
| Any release + ICSP/IDMS on OpenShift cluster | Works (outer decorator handles) | Works (unchanged — outer decorator runs first) |
| Any release + no overrides configured | Works | Works (unchanged — empty map, no replacements) |
| Component image tags in ImageStream | Overridden | Overridden (preserved) |

**Phase 2 (conditional — applied only if reproduction confirms failure):**

| Scenario | Hypothesized Before | After (if fix applied) | Condition |
|----------|---------------------|------------------------|-----------|
| Payload arch detection + `--registry-overrides` + non-OpenShift | `ValidReleaseImage=False`, reconciliation halted | Metadata provider applies override | R2 |
| GetDigest for Progressing + `--registry-overrides` + non-OpenShift | Stuck Progressing condition | Metadata provider applies override | R3 |
| Karpenter controller + any override config | Bare provider, no overrides | Full decorator chain | R4 |
| Karpenter ignition + `--registry-overrides` (native path) | `RegistryOverrides: nil` | Field populated | R5 |
| HCCO + `--registry-overrides` (native path) | `RegistryOverrides: nil` | Field populated with `o.registryOverrides` | R6 |

### Cache Impact

The `CachedProvider` sits below `RegistryMirrorProviderDecorator` in the chain. After the fix, the cache key changes from the original image URL to the overridden image URL. This is correct and desirable — it ensures cache entries match what was actually fetched. There is no risk of stale cache entries because the cache is populated per-process (in-memory map).

**Cache fragmentation under dual-decorator configs:** When both ICSP/IDMS (outer decorator) and `--registry-overrides` (inner decorator) are configured, the same logical image can be cached under different keys depending on which decorator path resolves it. For example, the outer decorator may rewrite `quay.io/foo` to `mirror-a.example.com/foo`, while the inner decorator would rewrite it to `mirror-b.example.com/foo`. This produces two cache entries for the same logical content. This is **harmless** — content is digest-pinned (so both entries are identical), the cache is in-memory with a 30-minute TTL, and the extra entry is a negligible memory cost. No mitigation is needed.

### Provider Chain Order

The fix does not change the provider chain order. `ProviderWithOpenShiftImageRegistryOverridesDecorator` (outer) still runs first, attempting ICSP/IDMS mirrors. If those fail or are not configured, it delegates to `RegistryMirrorProviderDecorator` (inner), which now applies `--registry-overrides` to both the lookup image and the result tags. The two override mechanisms are complementary and non-conflicting.

---

## Risks and Known Limitations

### Risk: Zero existing test coverage

`RegistryMirrorProviderDecorator` has **no unit tests** today. The only test that instantiates it (`TestProviderWithOpenShiftImageRegistryOverridesDecorator_Lookup` in `registry_image_content_policies_test.go:25`) passes empty `RegistryOverrides` and never exercises the override logic. This means the fix modifies untested code.

**Mitigation:** Unit tests MUST land with the fix, not as follow-up. AC5 explicitly mandates this.

### Known limitation: Overlapping override non-determinism

Go `map` iteration order is non-deterministic. If `RegistryOverrides` contains two entries that both match the same image (e.g., `quay.io` and `quay.io/openshift-release-dev`), the result varies between runs because `strings.Replace` is applied in arbitrary order.

This is a **pre-existing limitation** — the same non-determinism exists in the current tag post-processing loop (lines 36-39 of `registry_mirror_provider.go`). The fix inherits this behavior; it does not introduce new non-determinism. In practice, operators configure non-overlapping override entries (one source prefix per mirror), so this is unlikely to manifest.

**No mitigation in this epic.** If this needs to be addressed, it should be a separate effort that sorts map keys by specificity (longest prefix first) across both code paths.

### Known limitation: Override ordering inconsistency between release info and metadata paths

Fix 2 introduces a precedence reversal between the release info and metadata code paths:

| Path | First override applied | Second override applied |
|------|----------------------|------------------------|
| Release info (decorator chain) | ICSP/IDMS (outer `ProviderWithOpenShiftImageRegistryOverridesDecorator`) | `--registry-overrides` (inner `RegistryMirrorProviderDecorator`) |
| Image metadata (Fix 2) | `--registry-overrides` (string replacement at top of method) | ICSP/IDMS (`SeekOverride()` called after) |

**Concrete example of divergent behavior:** If an operator configures both ICSP/IDMS mapping `quay.io/foo` to `mirror-a/foo` and `--registry-overrides=quay.io/foo=mirror-b/foo`, then:
- Release info path: ICSP/IDMS rewrites to `mirror-a/foo` first; `--registry-overrides` does not match `mirror-a/foo`, so the result is `mirror-a/foo`.
- Metadata path: `--registry-overrides` rewrites to `mirror-b/foo` first; `SeekOverride()` does not match `mirror-b/foo`, so the result is `mirror-b/foo`.

This means the same image could be resolved to **different mirrors** depending on whether it goes through the release info path or the metadata path.

**On non-OpenShift clusters (the target scenario for this epic), this is irrelevant** — ICSP/IDMS is empty, so `SeekOverride()` is a no-op and only `--registry-overrides` has any effect. The inconsistency would only manifest on OpenShift management clusters that have **both** ICSP/IDMS policies **and** `--registry-overrides` configured simultaneously, where the same source registry is matched by both mechanisms. In practice, this dual-mechanism configuration is uncommon — ICSP/IDMS is the preferred mechanism on OpenShift clusters, and `--registry-overrides` is primarily used on non-OpenShift clusters where ICSP/IDMS is unavailable.

**No code change in this epic.** The reversal is an inherent consequence of bolting `--registry-overrides` support onto the metadata provider as a string replacement (the only viable approach without a larger refactor). The unified registry override abstraction (see Follow-up section) should resolve this by establishing a single, consistent ordering across all code paths.

### Known limitation: CLI `nodepool create` bare provider

`cmd/nodepool/core/create.go:82` instantiates `RegistryClientProvider` directly for version compatibility checking. This is a lower-priority production path (CLI validation, not reconciler hot path) and is **out of scope** for this epic.

**Mitigation:** A follow-up issue will be filed to wrap this usage in the decorator chain.

### Known limitation: E2E test bare providers

Five E2E test files instantiate `RegistryClientProvider` directly. These run in controlled environments with direct registry access and are **out of scope**.

---

## Security Considerations

### Pull Secret Handling

The pull secret is passed through the provider chain unchanged. The fix does not alter how pull secrets are handled — they are forwarded to the registry client for authentication against whichever registry URL is used (original or overridden). No new credential exposure.

### Registry Trust

The `--registry-overrides` flag is an operator-level configuration set by the cluster administrator. The fix does not introduce new override sources — it applies the **already-configured** overrides to an additional code path where they were missing. The trust model is unchanged: the operator administrator controls which registries are trusted as mirrors.

### Image Integrity

The override only replaces the registry portion of the image reference. The image digest (`@sha256:...`) is preserved, ensuring that the mirrored image has the same content as the original. Image verification (if any) continues to work against the digest.

### No New Attack Surface

- No new network connections — the fix redirects an existing registry pull to a different (configured) destination
- No new configuration inputs — uses the existing `--registry-overrides` flag
- No new permissions required — the mirror registry credentials should already be in the pull secret
- No credential leakage — pull secrets are not logged or exposed differently

---

## Follow-up: Unified Registry Override Abstraction

This epic fixes the immediate gaps with targeted patches. A follow-up epic should design a **unified override abstraction** to eliminate this class of bugs permanently.

### Problem Statement

Today there are two independent override mechanisms with different data structures, semantics, and coverage:

| | `--registry-overrides` | ICSP/IDMS |
|---|---|---|
| Data structure | `map[string]string` | `map[string][]string` |
| Semantics | Simple `strings.Replace` | Ordered mirror list, availability probing, caching, fallback |
| Release info path | `RegistryMirrorProviderDecorator` (inner decorator) | `ProviderWithOpenShiftImageRegistryOverridesDecorator` (outer decorator) |
| Image metadata path | **Not supported** (fixed by this epic, but bolted-on) | Native via `SeekOverride()` |
| Introduced | November 2021 | April 2023 |

Six separate wiring sites manually construct the decorator chain (HO, CPO `cpReleaseProvider`, CPO `userReleaseProvider`, HCCO, ignition-server, karpenter-operator). Three of these (CPO, HCCO, karpenter) merge `--registry-overrides` into the ICSP/IDMS map as a workaround for the missing native support — effectively abandoning the original mechanism's semantics.

### Why Unification Matters

- Every new image resolution code path must remember to wire **both** mechanisms, or it silently breaks on non-OpenShift clusters
- The merge-into-ICSP/IDMS workaround changes `--registry-overrides` semantics (adds mirror probing, 15s timeouts, availability caching) which operators may not expect
- The dual-decorator chain is hard to reason about — the interaction between outer and inner overrides is non-obvious
- The OLM catalogs have a **third** override entry point (via HCP annotation) that follows yet another pattern

### Scope for Follow-up Design

The follow-up design session should evaluate:

1. Whether to unify into a single abstraction or keep the mechanisms separate with consistent coverage
2. How to handle the semantic differences (simple replacement vs. mirror probing with fallback)
3. The serialization/deserialization round-trip through env vars and CLI flags across HO -> CPO -> HCCO -> ignition-server
4. Impact on the 6 wiring sites, 2 decorator types, mock/test infrastructure, and the OLM catalog path
5. Whether to split across multiple upstream PRs or land as one refactor

This should be its own epic with a dedicated design session — it is architecturally significant and touches 5 binaries across ~15 files.
