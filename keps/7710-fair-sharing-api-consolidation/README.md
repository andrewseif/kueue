# KEP-7710: Fair Sharing API Consolidation

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Cluster operator enabling AFS on a new ClusterQueue](#story-1-cluster-operator-enabling-afs-on-a-new-clusterqueue)
    - [Story 2: Existing user with deprecated admissionScope](#story-2-existing-user-with-deprecated-admissionscope)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Changes](#api-changes)
    - [v1beta2 (hub)](#v1beta2-hub)
    - [v1beta1](#v1beta1)
  - [Conversion Between v1beta1 and v1beta2](#conversion-between-v1beta1-and-v1beta2)
  - [Backward-Compatibility Fallback in Consumers](#backward-compatibility-fallback-in-consumers)
  - [Test Plan](#test-plan)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Summary

This KEP consolidates the Fair Sharing API surface on `ClusterQueue` by moving
the admission-fair-sharing configuration from the top-level
`spec.admissionScope` field into `spec.fairSharing.admissionFairSharing`. The
resulting layout mirrors the existing nesting on the status side
(`status.fairSharing.admissionFairSharingStatus`) so that all fair-sharing
configuration and status live under a single `fairSharing` field on both
`spec` and `status`.

The deprecated `spec.admissionScope` field is retained on both `v1beta1` and
`v1beta2` for backward compatibility and will be removed only when the API is
promoted past `v1beta2`, in accordance with Kubernetes API deprecation policy
for beta fields.

## Motivation

Fair sharing in Kueue is configured in two related places on `ClusterQueue`:

- `spec.fairSharing` configures the cohort-level fair-share weight.
- `spec.admissionScope` configures the admission-fair-sharing mode (the
  per-LocalQueue usage-based ordering of workloads).

Although these two settings are conceptually part of the same "fair sharing"
feature surface, they live at different levels of the spec. On the status side
the corresponding information is already nested: `status.fairSharing.weightedShare`
and `status.fairSharing.admissionFairSharingStatus`.

This asymmetry makes the API harder to discover, document, and reason about.
Users have to learn that fair-sharing-related configuration is split between
two sibling fields, while the corresponding status is grouped under one. The
inconsistency is reflected in user-facing documentation, examples, and the
generated API reference.

### Goals

- Nest `AdmissionFairSharing` under `FairSharing` on `v1beta2.ClusterQueueSpec`
  so spec and status both have a single `fairSharing` grouping.
- Preserve full backward compatibility for clusters and manifests that use the
  existing `spec.admissionScope` field on both `v1beta1` and `v1beta2`.
- Provide round-trip conversion between `v1beta1` and `v1beta2` such that the
  effective AFS configuration is preserved regardless of which field a user
  sets.
- Update Kueue's internal consumers of the AFS configuration to read from the
  new field first and fall back to `admissionScope` when needed.
- Update documentation and examples to use the new, consolidated field.

### Non-Goals

- Removing `spec.admissionScope` from `v1beta1` or `v1beta2`. Per Kubernetes
  API conventions, beta fields are not removed without an API version
  promotion. Removal is deferred to the next API version (e.g. `v1`).
- Changing the semantics of admission fair sharing itself. Modes,
  configuration, and behavior remain identical; only the location of the
  configuration in the spec changes.
- Changing the cohort-level fair-sharing weight semantics or the
  `status.fairSharing` shape.

## Proposal

Move the admission-fair-sharing configuration into the existing `FairSharing`
struct on the spec, behind a new optional field `admissionFairSharing`. Keep
the existing `admissionScope` field on `ClusterQueueSpec` as a deprecated
alias. All Kueue components that consume the configuration are taught to read
the new field first and fall back to the deprecated field for backward
compatibility.

### User Stories

#### Story 1: Cluster operator enabling AFS on a new ClusterQueue

A cluster operator creating a new `ClusterQueue` writes a single
`fairSharing` block that contains both the cohort weight and the
admission-fair-sharing mode:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: shared-queue
spec:
  fairSharing:
    weight: "1"
    admissionFairSharing:
      mode: UsageBasedAdmissionFairSharing
  resourceGroups:
  - ...
```

The configuration is grouped, easier to read, and matches the layout of
`status.fairSharing`.

#### Story 2: Existing user with deprecated admissionScope

An existing user has manifests using `spec.admissionScope`:

```yaml
spec:
  admissionScope:
    admissionMode: UsageBasedAdmissionFairSharing
```

These manifests continue to work without changes. Kueue's controllers and
schedulers read the deprecated field as a fallback when
`fairSharing.admissionFairSharing` is not set. Conversion between API versions
preserves the user's intent.

### Notes/Constraints/Caveats

- The `AdmissionFairSharing` type is added to `v1beta2.FairSharing` only. In
  `v1beta1.FairSharing` the field is not added; instead, conversion routes
  AFS configuration between `v1beta1.spec.admissionScope` and
  `v1beta2.spec.fairSharing.admissionFairSharing`.
- Because `v1beta2.FairSharing` and `v1beta1.FairSharing` no longer have an
  identical memory layout, the generated conversion code can no longer use an
  `unsafe.Pointer` cast for the `FairSharing` field in `ClusterQueueSpec`,
  `CohortSpec`, and `LocalQueueSpec`. The conversion is regenerated to do a
  proper field-by-field copy via the generated `autoConvert_*_FairSharing_*`
  helpers, and a manual `Convert_v1beta2_FairSharing_To_v1beta1_FairSharing`
  function is provided because the generator cannot fully auto-generate it
  (the v1beta2 `AdmissionFairSharing` field has no peer in v1beta1).

### Risks and Mitigations

- **Risk: silent data loss on round-trip conversion.** If a user submits a
  `v1beta1` `ClusterQueue` with `admissionScope` set, the v1beta2 view of the
  object must surface the same configuration under
  `spec.fairSharing.admissionFairSharing`, and vice versa.
  *Mitigation:* Conversion routines explicitly map between the two locations.
  Unit tests for `ClusterQueueConvertTo`/`ConvertFrom` verify round-trip
  preservation, including the case where both fields are present.

- **Risk: inconsistent behavior between Kueue's queue cache and scheduler
  cache.** The two caches independently read the configuration when building
  their in-memory representation of a ClusterQueue.
  *Mitigation:* Both caches share the same fallback logic — prefer
  `spec.fairSharing.admissionFairSharing`, fall back to `spec.admissionScope`,
  otherwise nil. The fallback is centralized in `admissionFairSharingFromSpec`
  in the queue cache, and mirrored by an equivalent `switch` in the scheduler
  cache.

- **Risk: API breakage for clients still using the v1beta1 / v1beta2
  `admissionScope` field.**
  *Mitigation:* The field is kept and marked `Deprecated:` in godoc. It is
  scheduled for removal only at the next API promotion, in line with
  Kubernetes' deprecation policy for beta APIs.

## Design Details

### API Changes

#### v1beta2 (hub)

Add a new optional field to `FairSharing`:

```go
type FairSharing struct {
    Weight *resource.Quantity `json:"weight,omitempty"`

    // admissionFairSharing contains the properties of the ClusterQueue
    // when participating in AdmissionFairSharing.
    // +optional
    AdmissionFairSharing *AdmissionFairSharing `json:"admissionFairSharing,omitempty"`
}

type AdmissionFairSharing struct {
    // +kubebuilder:validation:Enum=UsageBasedAdmissionFairSharing;NoAdmissionFairSharing
    // +required
    Mode AdmissionMode `json:"mode"`
}
```

The existing `ClusterQueueSpec.AdmissionScope` field is retained and marked
deprecated:

```go
// admissionScope indicates whether ClusterQueue uses the Admission Fair Sharing.
//
// Deprecated: Use FairSharing.AdmissionFairSharing instead.
// +optional
AdmissionScope *AdmissionScope `json:"admissionScope,omitempty"`
```

#### v1beta1

`v1beta1.FairSharing` is **not** extended with `AdmissionFairSharing`. The
existing `v1beta1.ClusterQueueSpec.AdmissionScope` field remains and is
marked deprecated with the same godoc convention. Conversion is responsible
for routing AFS configuration between the two API versions.

### Conversion Between v1beta1 and v1beta2

In `Convert_v1beta1_ClusterQueueSpec_To_v1beta2_ClusterQueueSpec`, after the
generated `autoConvert` runs, if `in.AdmissionScope` is set, copy its mode
into `out.FairSharing.AdmissionFairSharing` (allocating
`out.FairSharing` if necessary).

In `Convert_v1beta2_ClusterQueueSpec_To_v1beta1_ClusterQueueSpec`, after
`autoConvert` runs, if `in.FairSharing != nil` and
`in.FairSharing.AdmissionFairSharing != nil`, copy its mode into
`out.AdmissionScope`.

Because the v1beta1 and v1beta2 `FairSharing` structs no longer have
identical layouts, the regenerated conversion code calls
`autoConvert_v1beta{1,2}_FairSharing_To_v1beta{2,1}_FairSharing` instead of
casting via `unsafe.Pointer`. A manual
`Convert_v1beta2_FairSharing_To_v1beta1_FairSharing` is provided in
`apis/kueue/v1beta1/clusterqueue_conversion.go` because the conversion
generator cannot fully auto-generate it.

### Backward-Compatibility Fallback in Consumers

Two cache paths read the AFS configuration when building in-memory state:

- `pkg/cache/queue/cluster_queue.go` (`admissionFairSharingFromSpec`)
- `pkg/cache/scheduler/clusterqueue.go` (`updateClusterQueue`)

Both implement the same precedence:

1. If `spec.fairSharing.admissionFairSharing` is set, use it.
2. Otherwise, if `spec.admissionScope` is set, synthesize an
   `AdmissionFairSharing` from `admissionScope.admissionMode`.
3. Otherwise, leave the configuration unset.

The fallback path uses `//nolint:staticcheck` because it intentionally reads
the deprecated field.

### Test Plan

[X] I/we understand the owners of the involved components may require updates
to existing tests to make this code solid enough prior to committing the
changes necessary to implement this enhancement.

#### Unit tests

- `apis/kueue/v1beta1`: `TestClusterQueueConvertTo` /
  `TestClusterQueueConvertFrom` verify round-trip preservation of the AFS
  configuration regardless of which field (`admissionScope` or
  `fairSharing.admissionFairSharing`) was originally set.
- `pkg/cache/queue`: tests covering `admissionFairSharingFromSpec` exercise
  all three precedence cases.
- `pkg/cache/scheduler`: tests covering `updateClusterQueue` exercise the
  same three cases.

#### Integration tests

- ClusterQueue admission-fair-sharing integration tests are extended to run
  with manifests that use `spec.fairSharing.admissionFairSharing`. Existing
  tests that use the deprecated `spec.admissionScope` field are retained to
  verify the fallback path.

#### e2e tests

- No new e2e tests are required; existing AFS e2e coverage is sufficient
  because the underlying scheduling behavior is unchanged. Documentation
  examples are updated to use the new field.

### Graduation Criteria

This change is shipped with the existing AdmissionFairSharing feature, which
is already beta and enabled by default. The change is a structural API
consolidation rather than a new feature, so it inherits the maturity of
the feature it consolidates.

- **Beta (v0.18):** New field available on `v1beta2`, deprecated field
  retained, conversion and fallback in place, documentation updated.
- **GA / v1 promotion:** Remove `spec.admissionScope` from the API.

## Implementation History

- 2026-04-05: Initial PR opened (#7710).
- 2026-05-12: KEP drafted; conversion bug found and fixed (queue cache
  fallback to deprecated `admissionScope`); merge with main; documentation
  updated for English and Chinese sites.

## Drawbacks

- Carrying the deprecated `admissionScope` field on both `v1beta1` and
  `v1beta2` adds a small amount of permanent surface area until the next
  API promotion.
- The conversion code becomes slightly less efficient because the
  `FairSharing` field can no longer be copied via `unsafe.Pointer`; a
  proper field-by-field copy is required.

## Alternatives

- **Remove `spec.admissionScope` outright in `v1beta2`.** Rejected because
  it would be a breaking change to a beta API, contrary to Kubernetes API
  deprecation policy. PR review feedback explicitly identified this as
  unacceptable.
- **Add `admissionFairSharing` to `v1beta1.FairSharing` as well.** Rejected
  because it would expand the deprecated `v1beta1` surface. The same effect
  is achieved by routing the configuration through conversion.
- **Leave the API as-is.** Rejected because the asymmetry between
  `spec.admissionScope` and `status.fairSharing.admissionFairSharingStatus`
  is confusing for users and makes the API harder to document.
