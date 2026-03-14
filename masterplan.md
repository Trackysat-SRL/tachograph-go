     Masterplan: Resolve Code-Level Issues

     Context

     During the AGENTS.md rewrite, deep codebase analysis revealed several
     categories of issues: duplicate test infrastructure, incomplete Gen2 VU
     support (parsing → anonymization → marshalling), a future-proofing gap in
     card type dispatch, unimplemented certificate stubs, and a deferred
     UseRawData=false marshal path. This plan addresses them in dependency
     order, staged so each stage is independently shippable.

     Key Insight: Gen2 VU Parsing is the Bottleneck

     The Gen2 VU unmarshal functions (overview_gen2_v1.go,
     technical_data_gen2_v1.go,
     etc.) store raw_data but skip over record arrays without extracting semantic
     fields. This means:

     - Gen2 anonymization can't replace PII — the fields aren't populated
     - Gen2 semantic marshalling can't work — there's nothing to marshal from
     - Both depend on completing Gen2 parsing first

     The correct dependency chain is: parse → anonymize → semantic marshal.

     ---
     Stage 1: Consolidate duplicate test helpers

     Why: Reduces maintenance burden before we add more tests in later stages.

     Scope: Extract 3 identical functions from card and vu test helpers into
     a shared package.

     Files:
     - Create internal/testutil/golden.go (exported, not _test.go, so importable)
     - Edit internal/card/testdata_helpers_test.go — replace 3 functions with
     imports
     - Edit internal/vu/testdata_helpers_test.go — replace 3 functions with
     imports

     Functions to extract:
     - ReadHexdump(path string) ([]byte, error)
     - GoldenJSONPath(hexdumpPath string) string
     - LoadOrCreateGolden(t *testing.T, update bool, message proto.Message, goldenPath string)

     Keep in place: findHexdumpFiles() — different signatures per package
     (card: 3 enum params, vu: 1 enum param). Not worth abstracting.

     Note: *update flag stays per-package (Go test flag registration is
     per-binary). Pass as bool param to LoadOrCreateGolden.

     Verify: ./tools/mage test

     ---
     Stage 2: Gen2 VU semantic parsing

     Why: This is the bottleneck — all subsequent Gen2 work depends on it.

     Scope: Extend Gen2 VU unmarshal functions to extract semantic fields from
     record arrays instead of just skipping them.

     Files (7 VU Gen2 files):
     - internal/vu/overview_gen2_v1.go
     - internal/vu/overview_gen2_v2.go
     - internal/vu/technical_data_gen2_v1.go
     - internal/vu/technical_data_gen2_v2.go
     - internal/vu/events_faults_gen2_v1.go
     - internal/vu/events_faults_gen2_v2.go
     - internal/vu/detailed_speed_gen2.go

     Approach:
     - Each file already has a skipRecordArray loop that validates structure
     - Replace skip with actual parsing: read record array header (record count +
     record size), then iterate and unmarshal each record using existing dd-level
     unmarshal functions
     - Reference: Gen1 implementations in the same package show the target pattern
     (e.g., overview_gen1.go fully parses all fields)
     - The RecordArray binary format: [2-byte record count][2-byte record size][records...]

     Testing:
     - Update golden files for existing Gen2 hexdump fixtures
     (go test -update ./internal/vu/...)
     - Round-trip: unmarshal → unparse → compare
     - For record types without test data, add synthetic byte-slice tests per spec

     Verify: ./tools/mage test

     ---
     Stage 3: Gen2 VU anonymization

     Why: Now that semantic fields are populated (Stage 2), we can anonymize
     them.

     Scope: Implement full semantic anonymization for 5 Gen2 VU types.

     Files:
     - internal/vu/technical_data_gen2_v1.go — anonymizeTechnicalDataGen2V1
     - internal/vu/technical_data_gen2_v2.go — anonymizeTechnicalDataGen2V2
     - internal/vu/events_faults_gen2_v1.go — anonymizeEventsAndFaultsGen2V1
     - internal/vu/events_faults_gen2_v2.go — anonymizeEventsAndFaultsGen2V2
     - internal/vu/detailed_speed_gen2.go — anonymizeDetailedSpeedGen2

     Approach:
     - Follow Gen1 anonymize pattern (e.g., anonymizeTechnicalDataGen1 in
     technical_data_gen1.go): clone proto, anonymize PII fields (VIN, VRN,
     sensor serial numbers, card numbers, holder names), clear raw_data
     - Use existing dd.AnonymizeOptions helpers for shared types
     - ClearRawData() after anonymization — the raw bytes would contain
     unanonymized PII

     Testing:
     - Unit tests: anonymize, verify PII fields replaced, verify raw_data cleared
     - Round-trip: anonymize → marshal (requires raw_data painting to work from
     semantic fields — covered by Stage 4, or test with raw_data preserved
     temporarily)

     Verify: ./tools/mage test

     ---
     Stage 4: Gen2 VU semantic marshalling

     Why: Enables marshalling Gen2 data without raw_data (needed after
     anonymization clears it, and for constructed-from-scratch data).

     Scope: Implement 7 Gen2 VU marshal functions that currently error without
     raw_data.

     Files (same 7 as Stage 2):
     - internal/vu/overview_gen2_v1.go — MarshalOverviewGen2V1
     - internal/vu/overview_gen2_v2.go — MarshalOverviewGen2V2
     - internal/vu/technical_data_gen2_v1.go — MarshalTechnicalDataGen2V1
     - internal/vu/technical_data_gen2_v2.go — MarshalTechnicalDataGen2V2
     - internal/vu/events_faults_gen2_v1.go — MarshalEventsAndFaultsGen2V1
     - internal/vu/events_faults_gen2_v2.go — MarshalEventsAndFaultsGen2V2
     - internal/vu/detailed_speed_gen2.go — MarshalDetailedSpeedGen2

     Approach:
     - Keep raw_data painting path (existing — return raw_data when present)
     - Add semantic path as fallback: construct RecordArray binary format from
     proto fields using dd-level marshal functions
     - RecordArray layout: [2-byte count][2-byte size][count * size bytes of records][signature]
     - Reference: the unmarshal functions (Stage 2) serve as inverse specification

     Testing:
     - Round-trip: unmarshal hexdump → clear raw_data → marshal → compare to
     original bytes
     - Anonymize round-trip: unmarshal → anonymize (clears raw_data) → marshal →
     unmarshal → verify anonymized fields

     Verify: ./tools/mage test

     ---
     Stage 5: AppID V2 card type dispatch

     Why: Future-proofing for workshop/company/control card support.

     Scope: Make unmarshalApplicationIdentificationV2 dispatch by card type
     instead of hard-coding DRIVER_CARD.

     Files:
     - internal/card/application_identification_v2.go — add cardType param to
     unmarshal, dispatch to correct sub-message
     - internal/card/driver_card_file.go — update call site (~line 541) to pass
     cardv1.CardType_DRIVER_CARD

     Note: Binary format is always 4 bytes regardless of card type (the ASN.1
     SEQUENCE is identical). The difference is semantic: company/control cards only
     use vuConfigurationLengthRange, the other 3 fields are always zero. So the
     change is about setting the correct card_type and populating the right
     sub-message.

     Testing:
     - Existing golden tests must pass (driver card path unchanged)
     - Add table-driven tests for all 4 card types with synthetic 4-byte inputs
     - Round-trip for each card type

     Verify: ./tools/mage test

     ---
     Stage 6: Certificate client stubs

     Why: Cleanup — remove not implemented errors from production code.

     Scope: Implement GetRootCertificate() and GetEccRootCertificate() in
     internal/cert/client.go.

     Files:
     - internal/cert/client.go — lines 28-37
     - Possibly internal/cert/embedded.go — check if root certs are already
     embedded

     Approach:
     - Check whether root certificates are available in the embedded store
     (certcache) or need HTTP fetch from DTCJRC
     - Follow the pattern of GetRsaCertificate/GetEccCertificate (index
     lookup → HTTP GET → unmarshal)
     - Root certs may be simpler since they're well-known (single cert per
     generation, not per-country)

     Testing:
     - Unit test with mock HTTP client
     - Integration test against embedded cert store if applicable

     Verify: ./tools/mage test

     ---
     Deferred: UseRawData=false

     Why deferred: ~100 marshal functions need modification, no current demand,
     and Stage 4 partially addresses the need for Gen2. Revisit when there's a
     concrete use case for constructing tachograph data purely from semantic fields
     without ever having raw bytes.

     ---
     Execution Order

     Stage 1 (test helpers)     ─┐
     Stage 5 (AppID V2 dispatch) ├─ independent, can run in parallel
                                 │
     Stage 2 (Gen2 parsing)     ─┘
         │
         ▼
     Stage 3 (Gen2 anonymization)
         │
         ▼
     Stage 4 (Gen2 semantic marshal)
         │
         ▼
     Stage 6 (cert stubs)       ─── independent, anytime

     Verification (end-to-end)

     After all stages:
     ./tools/mage build    # full CI: download → generate → lint → test → tidy → cli
     → diff
     go test -update ./... # refresh all golden files
     ./tools/mage build    # verify golden updates don't cause further diffs

     "/plan open" to edit this plan in Nvim
