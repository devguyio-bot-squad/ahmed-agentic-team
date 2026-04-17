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

All callers pass the full decorator chain, so the fix is transparent:

| Caller | File | Provider Type |
|--------|------|---------------|
| HostedClusterReconciler | `hostedcluster_controller.go:672` | Has overrides |
| NodePoolReconciler | `nodepool_controller.go:980` | Has overrides |
| HAProxy reconciler | `haproxy.go:66` | Has overrides |
| HostedClusterSizing reconciler | `hostedclustersizing_controller.go:76` | Has overrides |

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
2. **Multiple matching overrides:** `strings.Replace` with count=1 ensures only the first match is replaced per override entry. The `for` loop over `RegistryOverrides` applies all matching entries sequentially — same behavior as the existing tag post-processing.
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

**Given** the modified `RegistryMirrorProviderDecorator.Lookup()` function
**When** unit tests are executed
**Then** tests verify:
- Override is applied to the image passed to the delegate
- Override is NOT applied when no matching entry exists
- Component tag post-processing still works
- Multiple overrides are applied correctly

### AC6: ICSP/IDMS decorator interaction preserved

**Given** an OpenShift management cluster with ICSP/IDMS configured
**And** `--registry-overrides` also configured
**When** the decorator chain processes a release image lookup
**Then** ICSP/IDMS overrides are attempted first (outer decorator), and `--registry-overrides` are applied to the image before the inner delegate call — both layers work together without conflict

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

### Provider Chain Order

The fix does not change the provider chain order. `ProviderWithOpenShiftImageRegistryOverridesDecorator` (outer) still runs first, attempting ICSP/IDMS mirrors. If those fail or are not configured, it delegates to `RegistryMirrorProviderDecorator` (inner), which now applies `--registry-overrides` to both the lookup image and the result tags. The two override mechanisms are complementary and non-conflicting.

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
