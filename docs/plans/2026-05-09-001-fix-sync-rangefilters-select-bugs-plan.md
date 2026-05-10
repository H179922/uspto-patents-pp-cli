---
title: "fix: Sync ID extraction, range-filter help text, and --select auto-unwrap"
type: fix
status: active
date: 2026-05-09
---

# fix: Sync ID extraction, range-filter help text, and --select auto-unwrap

## Summary

Three bugs found during live testing of the USPTO Patents CLI prevent core features from working: sync stores zero records because the ID extractor doesn't know the USPTO-specific field names, the `--range-filters` flag gives no format guidance so users always get HTTP 400, and `--select` requires knowledge of the API's internal wrapper key names to extract nested fields. This plan fixes all three so sync, local search/analytics/related, date-filtered queries, and field selection work out of the box.

---

## Problem Frame

During a dogfood session querying toddler-related patents, three issues surfaced:

1. `sync` fetches data successfully but discards every record because `extractID()` only checks generic field names (`id`, `name`, `uuid`, etc.) and the USPTO API uses domain-specific names (`applicationNumberText`, `trialNumber`, `appealNumber`, `petitionDecisionRecordIdentifier`, `productIdentifier`). This breaks `related`, `analytics`, and local `search` — the entire local data layer is non-functional.

2. `patent list-applications --range-filters 'filingDate:2024-11-09~2025-05-09'` returns HTTP 400. The CLI passes the string through verbatim, but the help text doesn't document the API's expected format: `applicationMetaData.filingDate 2024-11-09:2025-05-09` (space-separated field, colon-separated range). Users have to guess.

3. `--select applicationMetaData.inventionTitle` on `patent list-applications` returns empty output because the raw response is wrapped in `{"patentFileWrapperDataBag": [...]}` and `filterFields` operates on the top-level keys. Users must write the full path including the wrapper key, which they cannot reasonably know.

---

## Requirements

- R1. `sync` must store records for all USPTO resources that have identifiable items
- R2. `--range-filters` help text must document the expected format with examples
- R3. `--select` must auto-descend into single-child wrapper arrays so users can specify paths relative to the item, not the API envelope

---

## Scope Boundaries

- The `patent-applications-search-download` resource returns items with NO unique identifier — only `applicationMetaData` containing `inventionTitle`, `filingDate`, `applicationStatusDescriptionText`. This resource cannot be synced with the current download endpoint. Noted as a known limitation, not a bug to fix here.
- The `--select` auto-unwrap is scoped to this CLI. The same issue exists in the OpenFDA CLI and likely all generated CLIs — fixing the generator is a separate cross-cutting change.
- The three portfolio bugs found during testing (missing `patentFileWrapperDataBag` key, unquoted assignee name, missing `applicationStatusDescriptionText` field) were already fixed in-session and are not part of this plan.

### Deferred to Follow-Up Work

- Generator-level fix for `--select` auto-unwrap: generator issue on `mvanhorn/cli-printing-press`
- Sync for `patent-applications-search-download`: requires either switching to the non-download search endpoint (which includes `applicationNumberText`) or using composite keys

---

## Context & Research

### Relevant Code and Patterns

- `internal/cli/sync.go` lines 843-890: `resourceIDFieldOverrides` map (only 1 entry) and `extractID()` function with generic fallback list
- `internal/cli/sync.go` lines 513-563: `extractPageItems()` — correctly handles USPTO's non-standard wrapper keys via single-array fallback
- `internal/cli/helpers.go` lines 418-484: `filterFields()` and `filterFieldsRec()` — supports dot-notation and array descent, but operates on the raw response envelope
- All `patent_list-*.go` files: pass `--range-filters` string directly as `rangeFilters` query parameter with no format guidance

### API Response Structure (verified live)

| Resource | Wrapper key | Item-level ID field |
|----------|-------------|-------------------|
| `patent-applications-search-download` | `patentdata` | **None** (only nested `applicationMetaData`) |
| `patent-trials-proceedings-search-download` | `patentTrialData` | `trialNumber` |
| `patent-trials-decisions-search-download` | `patentTrialData` | `trialNumber` |
| `patent-trials-documents-search-download` | `patentTrialData` | `trialNumber` |
| `patent-appeals-decisions-search-download` | `patentTrialData` | `appealNumber` |
| `patent-interferences-decisions-search-download` | *(already mapped)* | `interferenceNumber` |
| `petition-decisions-search-download` | `petitionDecisionData` | `petitionDecisionRecordIdentifier` |
| `datasets` | `bulkDataProductBag` | `productIdentifier` |

---

## Key Technical Decisions

- **Add per-resource ID overrides rather than extending the generic fallback list**: The generic list (`id`, `name`, `uuid`, etc.) is a safety net for APIs that follow common conventions. USPTO's field names are domain-specific and should go in the override map where they're explicitly tied to their resource. Adding `trialNumber` to the generic list would cause false matches in other CLIs.

- **Document the range-filter format rather than inventing a structured CLI format**: The CLI is a passthrough for the API's query parameter contract. Inventing a CLI-side format (e.g., `--range-filters field=X,from=Y,to=Z`) creates a translation layer that must track API changes. Documenting the API's actual format with examples is simpler and more durable. The POST `create-applications` command already offers structured JSON for users who prefer it.

- **Auto-unwrap single-child wrapper objects in `filterFields`**: When the top-level JSON is an object with exactly one key whose value is an array, `filterFields` should descend into that array before applying path filters. This matches user mental model (users think about item fields, not API envelope structure). The unwrap only fires when the path doesn't match any top-level key, preserving backward compatibility.

---

## Implementation Units

- U1. **Add sync ID field overrides for all USPTO resources**

**Goal:** Sync stores records for 7 of 8 download resources (all except `patent-applications-search-download` which lacks an ID field entirely).

**Requirements:** R1

**Dependencies:** None

**Files:**
- Modify: `internal/cli/sync.go`

**Approach:**
- Add entries to `resourceIDFieldOverrides` for the 6 unmapped resources based on the verified API response structure above
- Leave `patent-applications-search-download` unmapped — it genuinely has no ID field in the download response

**Patterns to follow:**
- Existing `"patent-interferences-decisions-search-download": "interferenceNumber"` entry in the same map

**Test scenarios:**
- Happy path: Run `sync`, verify resources with ID overrides report stored records (not "0 items skipped")
- Happy path: After sync, `search "toddler" --data-source local` returns results from synced data
- Happy path: After sync, `related <trialNumber>` finds matches in local cache
- Edge case: `patent-applications-search-download` still reports "no extractable ID field" warning (expected — download endpoint lacks ID)
- Edge case: Re-sync (incremental) after initial sync does not create duplicates

**Verification:**
- `sync --json` output shows `stored > 0` for resources with ID overrides
- `analytics --type patent-trials-proceedings-search-download` returns grouped results

---

- U2. **Add format examples to --range-filters help text**

**Goal:** Users can successfully use `--range-filters` without guessing the format.

**Requirements:** R2

**Dependencies:** None

**Files:**
- Modify: `internal/cli/patent_list-applications.go`
- Modify: `internal/cli/patent_list-appeals.go`
- Modify: `internal/cli/patent_list-trials.go`
- Modify: `internal/cli/patent_list-trials-2.go`
- Modify: `internal/cli/patent_list-trials-3.go`
- Modify: `internal/cli/patent_list-interferences.go`
- Modify: `internal/cli/petition_list-decisions.go`
- Modify: `internal/cli/datasets_list.go`

**Approach:**
- Update the `--range-filters` flag description in every GET command that has one
- New description: `"Range filter: 'field from:to' (e.g., 'applicationMetaData.filingDate 2024-01-01:2025-01-01')"`
- Also update the `Example` string on `patent list-applications` to include a range-filter example

**Patterns to follow:**
- Existing flag help text style in the same files (e.g., `--sort` description)

**Test scenarios:**
- Happy path: `patent list-applications --help` shows the format example in `--range-filters` description
- Happy path: Running the documented example format returns results (not HTTP 400)

**Verification:**
- `--help` output for each modified command shows the new format example
- `patent list-applications --range-filters 'applicationMetaData.filingDate 2024-01-01:2025-05-09' --limit 5` returns results

---

- U3. **Auto-unwrap single-child wrapper arrays in filterFields**

**Goal:** `--select applicationMetaData.inventionTitle` works without users needing to know the API's wrapper key name.

**Requirements:** R3

**Dependencies:** None

**Files:**
- Modify: `internal/cli/helpers.go`

**Approach:**
- In `filterFields`, after parsing paths, check if the top-level JSON is an object
- If it's an object and none of the requested path heads match any top-level key, look for a single key whose value is an array
- If found, apply `filterFieldsRec` to that array instead of the top-level object, then re-wrap with the original key
- This preserves backward compatibility: if a user's path already matches a top-level key (like requesting `count` from `{"count": 300, "patentFileWrapperDataBag": [...]}`), it works as before
- Only the "no match" case triggers auto-descent

**Patterns to follow:**
- `extractPageItems()` in `sync.go` uses the same single-array-in-envelope detection pattern (lines 542-560)

**Test scenarios:**
- Happy path: `--select applicationMetaData.inventionTitle,applicationMetaData.filingDate` returns only those fields for each item
- Happy path: `--select count` on a response with `{"count": 300, "patentFileWrapperDataBag": [...]}` returns the count (no auto-unwrap — path matches top-level key)
- Edge case: `--select nonexistent` on a wrapped response returns empty items (not the whole response)
- Edge case: Response with two array keys (e.g., `{"a": [...], "b": [...]}`) does not auto-unwrap (ambiguous)
- Integration: `patent list-applications --q toddler --limit 3 --json --select applicationMetaData.inventionTitle` returns 3 items with only `inventionTitle`

**Verification:**
- The `--select` flag produces non-empty output for nested paths without requiring the wrapper key prefix

---

## System-Wide Impact

- **Interaction graph:** U1 changes `sync.go` which feeds the local SQLite store — `search --data-source local`, `analytics`, `related`, and `portfolio diff` all read from the store. After U1, these commands start returning real data.
- **Error propagation:** Sync continues to warn (not error) on `patent-applications-search-download` since that resource genuinely lacks an ID.
- **API surface parity:** U3 changes `filterFields` which is called by every command's JSON output path. The auto-unwrap must not break commands whose responses are already flat arrays or simple objects.
- **Unchanged invariants:** Provenance wrapping (`wrapWithProvenance`) and `compactFields` are not changed. `--select` continues to run before provenance wrapping in the command pipeline.

---

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Auto-unwrap in U3 could break commands with multiple top-level array keys | Only fires when exactly one key maps to an array AND no requested path matches a top-level key |
| Some trial resources share `trialNumber` across decisions and documents — potential ID collision during sync | Each resource type is stored in its own table in the SQLite store, so `trialNumber` uniqueness is per-resource |
| `datasets` items may change their structure across API versions | `productIdentifier` is the spec-documented primary key; unlikely to change without a version bump |
