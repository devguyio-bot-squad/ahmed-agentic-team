# Design: Fix Registry Override Not Applied When Extracting Release Metadata

**Epic:** #2 — Fix registry override not applied when extracting release metadata on non-OpenShift management clusters
**Upstream bug:** OCPBUGS-83564
**Date:** 2026-04-17

---

## Overview

On non-OpenShift management clusters (e.g., AKS in ARO-HCP), hosted cluster creation fails when using nightly OCP releases with `--registry-overrides` configured. The `RegistryMirrorProviderDecorator` applies registry overrides only to **component image tags** inside the returned `ImageStream`, but does **not** apply them to the **release image pullspec** before calling the delegate provider. This causes the system to query the original registry (`quay.io`) instead of the configured mirror (e.g., ACR), resulting in an `unauthorized` error for nightly releases that require authentication.

GA releases work because `quay.io/openshift-release-dev/ocp-release` is publicly accessible. Nightly releases (`ocp-release-nightly`) require authentication, exposing the bug.

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

### Systemic Gap Analysis

Implementation audit revealed that the `RegistryMirrorProviderDecorator.Lookup()` fix alone is **necessary but not sufficient** for the AKS + nightly scenario. The `--registry-overrides` mechanism has gaps across multiple code paths. These gaps do NOT affect ICSP/IDMS because that mechanism was designed later (April 2023 vs. November 2021) with correct override-before-access patterns from the start.

#### Why ICSP/IDMS Is Not Affected

The two override mechanisms are wired differently:

- **ICSP/IDMS** was built as a **two-pronged mechanism**: (1) `ProviderWithOpenShiftImageRegistryOverridesDecorator.Lookup()` applies overrides to the image *before* delegating in the release info path, and (2) `RegistryClientImageMetadataProvider` has its own `OpenShiftImageRegistryOverrides` field with every method calling `SeekOverride()` before registry access.

- **`--registry-overrides`** was only built into the release info path via `RegistryMirrorProviderDecorator`, and even there it was applied *after* the delegate call (to tags only). It was **never added** to `RegistryClientImageMetadataProvider`. Three components (CPO, HCCO, karpenter-operator) noticed this gap and worked around it by merging `--registry-overrides` values into the ICSP/IDMS-style `map[string][]string` — effectively abandoning the native mechanism and piggybacking on the ICSP/IDMS path. But this workaround only covers scenarios where ICSP/IDMS infrastructure is available.

#### Finding 1: `DetermineHostedClusterPayloadArch` — FATAL

**File:** `hypershift-operator/controllers/hostedcluster/hostedcluster_controller.go:1213`

`DetermineHostedClusterPayloadArch` calls `registryclient.IsMultiArchManifestList(ctx, hc.Spec.Release.Image, ...)` which passes the raw image through `RegistryClientImageMetadataProvider.GetManifest()`. This calls `SeekOverride()` for ICSP/IDMS but has no mechanism for `--registry-overrides`. On failure, the reconciler sets `ValidReleaseImage=False` and **returns the error** (line 1224), halting reconciliation entirely. **This blocks cluster creation before the `GetControlPlaneOperatorImage` fix is even reached.**

- **ICSP/IDMS:** Protected — `GetManifest()` calls `SeekOverride()` at `imagemetadata.go:262`
- **`--registry-overrides`:** Broken — `RegistryClientImageMetadataProvider` has no field for it

#### Finding 2: `GetDigest` for Progressing Condition — Operationally Fatal

**File:** `hypershift-operator/controllers/hostedcluster/hostedcluster_controller.go:1243`

`registryClientImageMetadataProvider.GetDigest(ctx, hcluster.Spec.Release.Image, pullSecretBytes)` passes the raw image. When this fails, the Progressing condition is set to `Reason: Blocked` permanently — monitoring alerts fire, upgrade orchestration stalls.

- **ICSP/IDMS:** Protected — `GetDigest()` calls `SeekOverride()` at `imagemetadata.go:216`
- **`--registry-overrides`:** Broken — same root cause as Finding 1

#### Finding 3: Karpenter Controller — FATAL (Neither Mechanism)

**File:** `karpenter-operator/main.go:130`

`ReleaseProvider: &releaseinfo.RegistryClientProvider{}` — a bare provider with **zero decorators**. Neither ICSP/IDMS nor `--registry-overrides` are applied. The `--registry-overrides` flag IS accepted (line 67) but never wired to this provider.

- **ICSP/IDMS:** Also broken — no decorator chain at all
- **`--registry-overrides`:** Broken — no decorator chain at all

#### Finding 4: Karpenter Ignition Controller — Broken Natively

**File:** `karpenter-operator/main.go:161-172`

The decorator chain has `RegistryOverrides: nil` on the inner `RegistryMirrorProviderDecorator`. The `--registry-overrides` flag values are merged into `imageRegistryOverrides` (the ICSP/IDMS-style `map[string][]string`, lines 154-159) instead of being wired to the inner decorator. This means `--registry-overrides` only works through the ICSP/IDMS path (outer decorator with mirror probing semantics), not through the native string-replacement path.

- **ICSP/IDMS:** Protected — outer decorator applies overrides including the merged flag values
- **`--registry-overrides`:** Broken natively — inner decorator receives `nil`, flag values are rerouted through ICSP/IDMS semantics

#### Finding 5: `RegistryClientImageMetadataProvider` — Systemic Root Cause

**File:** `support/util/imagemetadata.go:115-119`

The struct has `OpenShiftImageRegistryOverrides map[string][]string` but **no field for `--registry-overrides`** (`map[string]string`). Every method (`ImageMetadata`, `GetOverride`, `GetDigest`, `GetManifest`, `GetMetadata`) calls `SeekOverride()` which only processes ICSP/IDMS entries. This is the root cause of Findings 1 and 2 — any caller that resolves images through the metadata provider bypasses `--registry-overrides` entirely.

- **ICSP/IDMS:** Protected by design — every method calls `SeekOverride()`
- **`--registry-overrides`:** Not supported at all

#### Impact Matrix

| Code Path | Component | `--registry-overrides` | ICSP/IDMS | Severity |
|-----------|-----------|----------------------|-----------|----------|
| `Lookup()` (release info) | HO HC Reconciler | **Fixed** (decorator fix) | Works | Resolved |
| `DetermineHostedClusterPayloadArch` | HO HC Reconciler | **Broken** — no metadata support | Works | FATAL — blocks before fix is reached |
| `GetDigest` for Progressing | HO HC Reconciler | **Broken** — no metadata support | Works | Operationally fatal |
| Karpenter controller `Lookup` | Karpenter Operator | **Broken** — bare provider | **Also broken** | FATAL |
| Karpenter ignition `Lookup` | Karpenter Operator | Broken natively (`nil`) | Works (via workaround) | Broken on non-OpenShift |
| NodePool reconciler (all paths) | HO NodePool Reconciler | Works | Works | OK |
| CPO `cpReleaseProvider` | CPO | Works | Works | OK |
| CPO `userReleaseProvider` | CPO | `nil` (intentional) | Works | OK (by design) |
| Ignition server | Ignition Server | Works | Works | OK |

---

## Components and Interfaces

### Modified Components

#### Fix 1: `RegistryMirrorProviderDecorator.Lookup()` — `support/releaseinfo/registry_mirror_provider.go:26-46`

Apply `RegistryOverrides` string replacement to the `image` parameter **before** calling `p.Delegate.Lookup()`, in addition to the existing post-processing of component image tags.

#### Fix 2: `RegistryClientImageMetadataProvider` — `support/util/imagemetadata.go:115-119`

Add a `RegistryOverrides map[string]string` field. Apply simple string replacement to the image reference **before** calling `SeekOverride()` in each method (`ImageMetadata`, `GetOverride`, `GetDigest`, `GetManifest`, `GetMetadata`). This ensures `--registry-overrides` is applied in the image metadata path, resolving Findings 1 and 2.

#### Fix 3: Karpenter controller wiring — `karpenter-operator/main.go:127-131`

Replace the bare `&releaseinfo.RegistryClientProvider{}` with the full decorator chain, matching the pattern already used for the karpenter ignition controller at lines 161-172. This resolves Finding 3.

#### Fix 4: Karpenter ignition controller wiring — `karpenter-operator/main.go:169`

Wire `registryOverrides` to `RegistryMirrorProviderDecorator.RegistryOverrides` instead of (or in addition to) merging into the ICSP/IDMS map. This resolves Finding 4.

#### Fix 5: `RegistryClientImageMetadataProvider` wiring — all instantiation sites

Pass `registryOverrides` to the new `RegistryOverrides` field on `RegistryClientImageMetadataProvider` at each instantiation site:

| Site | File | Current | Fix |
|------|------|---------|-----|
| HO | `support/globalconfig/imagecontentsource.go:251` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| Karpenter | `karpenter-operator/main.go:174` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| CPO | `control-plane-operator/main.go:479` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| HCCO | `hostedclusterconfigoperator/cmd.go:277` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |
| Ignition server | `ignition-server/cmd/start.go:155` | ICSP/IDMS only | Add `RegistryOverrides: registryOverrides` |

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
| karpenter-operator Reconciler | `karpenter-operator/main.go:130` | **Production** — same bug on nightly releases | **In scope** — Fix 3 wraps in decorator chain |
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

### AC8: Image metadata path supports `--registry-overrides`

**Given** `RegistryClientImageMetadataProvider` configured with `--registry-overrides` mapping `quay.io/openshift-release-dev/ocp-release-nightly` to an accessible mirror
**When** `ImageMetadata()`, `GetDigest()`, or `GetManifest()` is called with a nightly release image
**Then** the override is applied to the image reference before registry access, and the call succeeds against the mirror

### AC9: `DetermineHostedClusterPayloadArch` succeeds on non-OpenShift clusters

**Given** a non-OpenShift management cluster with `--registry-overrides` configured for nightly images
**When** a HostedCluster is created with a nightly OCP release
**Then** `DetermineHostedClusterPayloadArch` resolves the payload architecture successfully without `ValidReleaseImage=False`

### AC10: Karpenter controller uses decorator chain

**Given** the karpenter-operator configured with `--registry-overrides`
**When** the karpenter controller reconciles a HostedControlPlane with a nightly release
**Then** the `ReleaseProvider.Lookup()` call applies registry overrides (not bare `RegistryClientProvider`)

### AC11: Karpenter ignition controller wires `--registry-overrides` natively

**Given** the karpenter-operator configured with `--registry-overrides`
**When** the karpenter ignition controller processes a release image lookup
**Then** `RegistryMirrorProviderDecorator.RegistryOverrides` is populated (not `nil`) and overrides are applied via native string replacement

---

## Impact on Existing System

### Blast Radius

The changes span two support packages and three wiring sites:

| Change | Files | Nature |
|--------|-------|--------|
| Fix 1: `RegistryMirrorProviderDecorator.Lookup()` | 1 file | Behavioral — apply override before delegation |
| Fix 2: `RegistryClientImageMetadataProvider` | 1 file | Structural — add `RegistryOverrides` field, apply in 5 methods |
| Fix 3: Karpenter controller wiring | 1 file | Wiring — replace bare provider with decorator chain |
| Fix 4: Karpenter ignition wiring | 1 file | Wiring — populate `RegistryOverrides` field |
| Fix 5: Metadata provider wiring | 5 files | Wiring — pass `registryOverrides` to metadata provider |

No interface changes, no new dependencies, no configuration changes. The `RegistryClientImageMetadataProvider` struct gains one new field — existing callers that don't set it get the zero value (`nil`), preserving current behavior.

### Behavioral Changes

| Scenario | Before | After |
|----------|--------|-------|
| Nightly release + `--registry-overrides` + non-OpenShift cluster | **FAILS** — pulls from original registry | **WORKS** — pulls from mirror |
| Payload arch detection + `--registry-overrides` + non-OpenShift | **FAILS** — `ValidReleaseImage=False`, reconciliation halted | **WORKS** — metadata provider applies override |
| GetDigest for Progressing + `--registry-overrides` + non-OpenShift | **FAILS** — stuck Progressing condition | **WORKS** — metadata provider applies override |
| Karpenter controller + any override config | **FAILS** — bare provider, no overrides | **WORKS** — full decorator chain |
| Karpenter ignition + `--registry-overrides` (native path) | **BROKEN** — `RegistryOverrides: nil` | **WORKS** — field populated |
| GA release + `--registry-overrides` (no matching entry) | Works (public access) | Works (unchanged — no override match) |
| Any release + ICSP/IDMS on OpenShift cluster | Works (outer decorator handles) | Works (unchanged — outer decorator runs first) |
| Any release + no overrides configured | Works | Works (unchanged — empty map, no replacements) |
| Component image tags in ImageStream | Overridden | Overridden (preserved) |

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
