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

---

## Components and Interfaces

### Modified Component

**`RegistryMirrorProviderDecorator.Lookup()`** — `support/releaseinfo/registry_mirror_provider.go:26-46`

This is the only function that needs modification. The fix applies `RegistryOverrides` string replacement to the `image` parameter **before** calling `p.Delegate.Lookup()`, in addition to the existing post-processing of component image tags.

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

### Direct `RegistryClientProvider` Usages (Same Bug Class — Out of Scope)

The following callers instantiate `RegistryClientProvider` directly, bypassing the decorator chain entirely. They suffer from the same class of bug (no registry overrides applied) but are out of scope for this epic, which targets the decorator chain fix.

| Caller | File | Risk | Scope Decision |
|--------|------|------|----------------|
| karpenter-operator Reconciler | `karpenter-operator/main.go:130` | **Production** — same bug on nightly releases | Out of scope — follow-up issue required |
| `hypershift create nodepool` CLI | `cmd/nodepool/core/create.go:82` | **Production** — CLI validation path | Out of scope — follow-up issue required |
| NodePool upgrade E2E test | `test/e2e/nodepool_upgrade_test.go:166` | Low — test infra | Out of scope |
| Karpenter E2E test | `test/e2e/karpenter_test.go:481` | Low — test infra | Out of scope |
| Karpenter CP upgrade E2E test | `test/e2e/karpenter_control_plane_upgrade_test.go:56` | Low — test infra | Out of scope |
| E2E version util | `test/e2e/util/version.go:55` | Low — test infra | Out of scope |
| E2E util | `test/e2e/util/util.go:3936` | Low — test infra | Out of scope |

**Note:** The karpenter-operator file has *two* `RegistryClientProvider` usages — line 130 (direct, no decorators) and line 165 (properly wrapped in the full chain). Only the direct usage at line 130 is affected.

**Follow-up:** A separate issue should be filed to wrap the two production-path direct usages (`karpenter-operator/main.go:130` and `cmd/nodepool/core/create.go:82`) in the decorator chain. The E2E test usages are lower priority as they run in controlled environments with direct registry access.

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

---

## Impact on Existing System

### Minimal Blast Radius

The change is confined to a single function: `RegistryMirrorProviderDecorator.Lookup()` in `support/releaseinfo/registry_mirror_provider.go`. No interface changes, no new dependencies, no configuration changes.

### Behavioral Changes

| Scenario | Before | After |
|----------|--------|-------|
| Nightly release + `--registry-overrides` + non-OpenShift cluster | **FAILS** — pulls from original registry | **WORKS** — pulls from mirror |
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

### Known limitation: Direct `RegistryClientProvider` usages

Seven call sites instantiate `RegistryClientProvider` directly, bypassing the decorator chain (see "Direct `RegistryClientProvider` Usages" table in Components and Interfaces). Two are production code paths (`karpenter-operator/main.go:130` and `cmd/nodepool/core/create.go:82`) with the same class of bug. These are **explicitly out of scope** for this epic.

**Mitigation:** A follow-up issue will be filed to wrap the two production-path direct usages in the decorator chain after this fix lands.

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
