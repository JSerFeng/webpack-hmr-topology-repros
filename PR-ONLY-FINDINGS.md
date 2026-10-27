# Historical Rspack PR findings: separate from upstream reproductions

These items came up while reviewing successive designs of [Rspack PR #15955](https://github.com/web-infra-dev/rspack/pull/15955). They are recorded here to preserve the full discussion inventory. **Their historical failing revisions have not been rebuilt in this task, and this table is not a claim that the current PR still has every defect.**

## Implementation-specific correctness findings

| Item | Mechanism / symptom | Relationship to the upstream demos |
| --- | --- | --- |
| Partial HMR payload does not establish enduring ownership | An install-only factory is registered and executed, but its ordinary chunk was never fully loaded. Once the loaded block stops referring to that chunk, later edits can be skipped. Reattaching the block can then skip an install-only payload because an older local factory remains. | Same missing-update invariant as demo 4, but the install-only array protocol is Rspack-PR-specific. |
| Ordinary and install-only payloads collide | The same module can arrive from different chunk/runtime comparisons as both a normal update and an install-only payload. A normal changed factory must win regardless of response arrival order. | No equivalent array protocol exists in the stock-webpack demos. |
| Removal confused chunk membership with runtime membership | A module moved out of a chunk but remained in the same runtime; incorrectly emitting it in manifest m disposed a still-needed module. | Compiler regression in an earlier PR iteration; different from demo 4's missed future update. |
| Factory hash compared across unlike runtime variants | Comparing a runtime-specific factory against all overlapping old runtime variants can classify unchanged code as changed, causing spurious unaccepted propagation. | Earlier PR comparison bug; not the topology-generated factory change isolated by demo 3. |
| Loaded block identity changes with import source location | An executed import's recorded block key can be lost when its location changes, even though a named physical loading group is still relevant. The subsequent update may omit a needed dependency. | Named-group identity was a separate PR refinement; no block-id protocol is added by these webpack demos. |
| Removed/reintroduced block keeps stale loaded state | A stale installedBlocks entry can eagerly download a reintroduced but uncalled branch, accumulate historical keys, or be resurrected by a late completion callback. | PR-specific state reconciliation; not proof that old module cache must be deleted. |
| Async Worker entry plans omitted | Traversing only normal entrypoints or filtering all non-initial groups misses a running Worker's startup dependency closure. | Demo 7 reproduces the general Worker dependency gap on webpack itself. |
| Entry/chunk ID changes and bookkeeping | Physical entry IDs and stable logical identities must not be conflated when selecting old loaded entries. | A review edge case; natural-ID-specific expansion was outside the agreed dev scope. |
| Concatenated block parent has no direct chunk membership | A block inside a concatenated inner module may survive codegen while a parent-membership lookup produces an empty map. | A historical PR-only configuration finding; HMR plus concatenation was excluded from the main design scope. |

## Design/performance concerns, not additional confirmed upstream bugs

- Re-emitting a full block-to-chunk map on every update increased dev transfer size; a runtime-module generate_update path was explored to emit deltas while keeping full initial output.
- Historical snapshot asset retention changed old-asset semantics and could accumulate state; it was removed from the desired approach.
- Updating a live block map immediately on receiving a manifest does not itself make ordinary chunk bytes match that version. Demo 6 isolates the remaining version gap.
- Runtime module custom-source fallback, empty-map/deletion transitions, failed-build baselines, and state preservation need coherent update handling; they should not be listed as bugs without a failing case.
- CSS insertion within a loading group and cross-group order are different guarantees. Cross-group ordering that normal loading never promised should not be presented as an HMR regression. Demo 8 focuses on the confirmed missing/new stylesheet problem.
- Supplemental JS loading does not automatically implement every CSS/Wasm/plugin ensure handler. Coverage is a separate scope question.
- HMR error, browser reload, and successful first-screen rendering are different observations. None of the Node demos counts browser reloads.

## Module lifetime question

The proposal to dispose a module when it loses all loaded-group owners is a possible new lifecycle policy, not an established fix in these repositories. Deleting cache alone leaves a stale factory usable; deleting factories also requires loading-state reconciliation. Retaining module instances instead requires continuing to deliver their updates and supplying new synchronous dependencies. Demos 4 and 5 provide separate concrete inputs for that discussion.
