# Camera-Ready Plan (post-acceptance, up to 4 pages)

## Priority 1: Expand Evaluation
- Add selectivity sensitivity experiment for the secondary index: vary the number of matching `ss_customer_sk` keys and report how scan reduction changes. Even 3-4 data points would strengthen the evaluation significantly.
- Report variance across runs (error bars or min/median/max) for both the secondary index and RLI benchmarks.

## Priority 2: Metadata Access Pattern Characterization
- Add a paragraph (ideally with a small table or inline numbers) breaking down production metadata access patterns: what fraction are point lookups vs. prefix scans vs. range scans?
- This turns the design claim ("optimized for point and prefix lookups") into an empirical finding.

## Priority 3: HFile Tradeoff Discussion
- Add a short paragraph discussing when HFile is *not* the right choice (e.g., analytical/columnar scans over metadata, bulk metadata exports).
- Shows intellectual honesty and helps readers understand the design space.

## Priority 4: Recovery Semantics
- One paragraph explaining what happens when a crash occurs during the atomic commit window (metadata table inconsistency detection and repair).
- Addresses a natural reviewer question about the parent-child timeline coupling under failures.

## Notes
- The extra page gives roughly 2 columns of space (~80 lines of body text).
- Priorities 1-2 are highest ROI for reviewer satisfaction.
- All additions should finally go into `main.tex`. But, first make revisions in `main_revised.tex` file.

----

Plan: FORMATS '26 Camera-Ready Revisions

     Context

     The FORMATS '26 paper "Metadata-as-Data in Apache Hudi: A Multi-Modal Index Substrate for Lakehouse Tables" has been accepted (3× Strong Accept). The camera-ready version
      allows 4 pages (up from 3), giving us ~2 extra columns to address reviewer feedback. The core paper is strong — revisions are about deepening the Discussion section and
     adding tradeoff analysis that reviewers found missing.

     Reviewer Comments — Consolidated

     Must-Address (raised by multiple reviewers)

     ┌─────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬──────────┐
     │ ID  │                                                      Comment                                                      │  Reviewers   │ Priority │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────┼──────────┤
     │ C1  │ DuckLake comparison: metadata in SQL DB vs self-managed HFile — a fundamental design trade-off                    │ R1-W3, R2-Q2 │ High     │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────┼──────────┤
     │ C2  │ Write/ingestion tradeoffs: does enabling indexing affect write speed, TPS, storage overhead?                      │ R2-W1, R2-Q4 │ High     │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────┼──────────┤
     │ C3  │ RUM conjecture: frame design decisions as informed trade-offs, not just "best choices"                            │ R4-W1        │ High     │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────┼──────────┤
     │ C4  │ Deeper cross-format comparison: where have Delta/Iceberg aligned and diverged? Can formats learn from each other? │ R1-W2, R4-W2 │ High     │
     └─────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴──────────┘

     Should-Address (single reviewer, good suggestions)

     ┌─────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────┬──────────────────────────────────┐
     │ ID  │                                                    Comment                                                    │ Reviewer │             Priority             │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┼──────────────────────────────────┤
     │ C5  │ Index granularity: can indexing target files, row groups, pages, rows? Is resolution builtin or extensible?   │ R2-Q3    │ Medium                           │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┼──────────────────────────────────┤
     │ C6  │ Non-co-located indexes: vector/bitmap where index doesn't co-locate with data — a challenge for the approach? │ R2-Q1    │ Medium                           │
     ├─────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┼──────────────────────────────────┤
     │ C7  │ Standardization: no specification changes proposed for cross-format standardization                           │ R1-W1    │ Low (acknowledge in future work) │
     └─────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────┴──────────────────────────────────┘

     Noted (no action needed — positive feedback)

     - R1-S1: Addresses real problem at scale; clean solution
     - R1-S2: Table 1 mapping to DB analogues is well thought
     - R2-S1: Indexing structures becoming important to table formats
     - R2-S2: Concurrent indexing addresses keeping indexes up to date
     - R2-S3: Design is novel, powerful, and simple
     - R4-S1: Hudi is fundamentally worth talking about
     - R4-S2: Components driving Hudi's performance are of interest to FORMATS

     ---
     Change Plan

     Overview: Where the Extra Page Goes

     The current paper is tight at 3 pages of content. The extra page (~2 columns) will be used primarily to expand the Discussion section (currently ~0.5 page → ~1.5 pages)
     into a substantive design-space analysis. Minor additions go into existing sections.

     Change 1: Expand Discussion Section — Design Trade-offs & Cross-Format Analysis

     Addresses: C1, C2, C3, C4
     File: main_revised.tex Section 6 (Discussion), lines 385–394
     Space: ~1.5 pages (up from ~0.5)

     Restructure the Discussion into subsections:

     1a. RUM Conjecture Framing (C3) — NEW paragraph, ~0.3 page

     Add a paragraph at the start of Discussion framing Hudi's design through the RUM conjecture (Athanassoulis et al., 2016): any index structure can optimize at most two of
     Read, Update, Memory. Hudi's metadata table makes an explicit RUM trade-off:
     - Optimizes Reads: HFile layout, prefix keys, multi-modal pruning → sub-millisecond lookups
     - Optimizes Updates: MoR with delta logs → metadata updates append without rewriting base files
     - Trades Memory (space): additional storage cost (record index adds ~7% for large tables, under 1% for small)

     This reframes the design as an informed trade-off, not just a claim of superiority. The MoR choice specifically favors R and U at the cost of M (compaction eventually
     reclaims space, but storage overhead is non-zero).

     1b. Write Performance & Storage Tradeoffs (C2) — NEW paragraph, ~0.3 page

     Add concrete tradeoff data after the RUM framing:
     - Storage overhead: record index adds ~7% of data size for the largest table (549 TB → 40 TB index); under 1% for small tables. Column stats and bloom filters are
     comparatively small.
     - Write latency impact: each data commit must also update the metadata table (transactional coupling). MoR design mitigates this — metadata updates are appended as delta
     logs, not rewritten. Background compaction amortizes the cost. The 72% write latency reduction from RLI (cited in §5) shows that for upsert workloads, the index lookup
     savings far exceed the maintenance cost.
     - Compaction overhead: metadata table compaction runs asynchronously, consuming background I/O. At Uber, compaction frequency is tuned per-table based on delta-log
     accumulation.

     1c. Deeper Cross-Format Comparison (C4) — EXPAND existing paragraph, ~0.4 page

     Expand the current Delta/Iceberg discussion (lines 387–389) with a comparison table or structured comparison:

     ┌──────────────────────┬──────────────────────────────────────────────────┬──────────────────────────────────────────────┬──────────────────────────────────────────────┐
     │   Design dimension   │                       Hudi                       │                  Delta Lake                  │                   Iceberg                    │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Metadata structure   │ Internal MoR table (partitioned)                 │ Transaction log + periodic Parquet           │ Manifest hierarchy + Puffin sidecars         │
     │                      │                                                  │ checkpoints                                  │                                              │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Statistics storage   │ Dedicated column_stats partition (HFile)         │ Embedded in checkpoint Parquet files         │ Per-file stats in manifest entries           │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Record-level index   │ record_index partition (hash-sharded HFile)      │ N/A                                          │ N/A (row lineage proposed in V3)             │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Bloom filters        │ bloom_filters partition                          │ N/A                                          │ Puffin blob (apache-datasketches)            │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Adding new index     │ New metadata-table partition (no protocol        │ Requires checkpoint schema change            │ New Puffin blob type (requires reader        │
     │ type                 │ changes)                                         │                                              │ support)                                     │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Lookup pattern       │ Point + prefix lookups via HFile block index     │ Full checkpoint scan or log replay           │ Manifest scan + Puffin file reads            │
     ├──────────────────────┼──────────────────────────────────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────┤
     │ Maintenance coupling │ Transactional (same-instant commit)              │ Implicit (stats derived from log)            │ Snapshot-level (manifest rewrite)            │
     └──────────────────────┴──────────────────────────────────────────────────┴──────────────────────────────────────────────┴──────────────────────────────────────────────┘

     Key insight to articulate: Delta and Iceberg embed statistics within their commit/manifest structures, which simplifies the architecture but makes extensibility harder
     (adding a new index type requires changing the commit format). Hudi's separate-but-coupled metadata table allows independent index evolution — the same MoR/HFile
     infrastructure serves all modalities.

     Note where formats align (all use per-file column stats for data skipping) and where they diverge (only Hudi supports record-level and secondary indexes as first-class
     metadata).

     1d. DuckLake Comparison (C1) — NEW paragraph, ~0.3 page

     Add after the cross-format comparison:

     DuckLake takes the metadata-as-data idea further by storing all metadata in a relational database (DuckDB or PostgreSQL) rather than in files on object storage. This
     represents a fundamental design trade-off:
     - Co-located file-based metadata (Hudi): no external dependency, same durability/replication as data, works in any object-store deployment, but requires purpose-built
     storage (HFile) and its own compaction. Self-contained but complex.
     - Externalized DB metadata (DuckLake): leverages SQL query capabilities, simpler metadata access patterns, and proven DB indexing — but introduces an external dependency,
      a separate failure domain, and a consistency boundary between the metadata DB and data files. Simpler for queries but more powerful, at the cost of operational coupling.

     The choice depends on deployment context: serverless and multi-cloud environments favor co-located metadata (no additional service); managed platforms with always-on
     infrastructure may prefer externalized metadata for its query flexibility.

     Add DuckLake citation to references.bib.

     Change 2: Index Granularity Clarification

     Addresses: C5
     File: main_revised.tex Section 3 (Multi-Modal Indexing), around line 220
     Space: ~2-3 sentences added to existing paragraph

     Add a clarifying sentence after the index substrate introduction (line 220):
     "The current granularity is per-file for column statistics and bloom filters, and per-record for the record-level and secondary indexes. Per-file granularity aligns with
     the immutable file-group model; finer-grained indexing (e.g., per-row-group) is an extensible design point since new granularities can be added as additional
     metadata-table partitions."

     Change 3: Non-Co-located Indexes Acknowledgment

     Addresses: C6
     File: main_revised.tex Section 7 (Conclusion), line 400
     Space: ~1-2 sentences added to future work

     Expand the future directions sentence to acknowledge the co-location challenge:
     "Future directions include bitmap and vector indexes — where index entries may not co-locate with data file groups, requiring the metadata table to serve as a standalone
     lookup structure rather than a per-file-group auxiliary — as well as cost-based index selection and cross-format index interoperability."

     Change 4: Minor — Update Page Limit Comment

     File: main_revised.tex line 2
     Change the comment from "3 pages excluding references" to "4 pages excluding references".

     ---
     Files to Create/Modify

     Important: main.tex must NOT be modified — it is the submitted version and should remain constant.

     ┌──────────────────┬───────────────────────────┬─────────────────────────────────────────────────────────────────────┐
     │       File       │          Action           │                             Description                             │
     ├──────────────────┼───────────────────────────┼─────────────────────────────────────────────────────────────────────┤
     │ main_revised.tex │ Create (copy of main.tex) │ Camera-ready version with all reviewer comments addressed           │
     ├──────────────────┼───────────────────────────┼─────────────────────────────────────────────────────────────────────┤
     │ references.bib   │ Edit                      │ Add DuckLake and RUM conjecture citations (shared by both versions) │
     └──────────────────┴───────────────────────────┴─────────────────────────────────────────────────────────────────────┘

     New References to Add

     1. RUM conjecture: Athanassoulis et al., "Designing Access Methods: The RUM Conjecture," EDBT 2016
     2. DuckLake: Raasveldt and Muhleisen, "DuckLake: SQL as a Lakehouse Format," 2025 (or appropriate citation — verify correct form)

     ---
     Verification

     1. Compile: latexmk -pdf main_revised.tex — must produce clean PDF
     2. Original unchanged: diff main.tex should show no modifications to the submitted version
     3. Page count: Verify content fits in 4 pages excluding references
     4. All reviewer comments addressed: Cross-check each C1–C7 against changes
     5. No new content that contradicts existing claims: Especially the tradeoff numbers (7% storage, 72% write latency reduction) must be consistent with §5
     6. ACM compliance: \Description on all figures, table captions above body, figure captions below body
     7. Citation accuracy: Verify DuckLake and RUM conjecture citations against DBLP/publisher
