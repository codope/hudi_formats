# Plan: CDMS 2026 Workshop Paper — Composable Lakehouse Metadata & Index Discovery Protocol

## Context

Sagar submitted a 4-page paper to FORMATS '26 (SIGMOD workshop) on "Metadata-as-Data in Apache Hudi: A Multi-Modal Index Substrate for Lakehouse Tables" — covering the metadata table architecture, multi-modal indexing, concurrent indexing protocol, and evaluation on TPC-DS/Uber production tables.

He now wants to write a **complementary but distinct** 6-page paper for **CDMS 2026** (Composable Data Management Systems, co-located with VLDB 2026, Sept 4, Boston). The paper combines two framings:

- **Hook (Framing 6+1):** XTable/UniForm create an *illusion* of interoperability by translating the easy 10% of metadata (schema, file listings) while the indexes and statistics delivering 10-100x performance gains remain format-specific and engine-specific.
- **Core contribution (Framing 3):** A concrete **Lakehouse Index Protocol** — analogous to LSP (Language Server Protocol) — for engines to discover available indexes, negotiate capabilities, and safely consume them.

**Research strategist assessment:** Low scooping risk. No one has published a formal index discovery protocol for lakehouses. The TUM "Active Data Lakes" paper (Jan 2026) overlaps in motivation but focuses on architecture, not protocols. Window is 6-12 months before someone else occupies this space.

---

## Paper Structure (6 pages + references, VLDB workshop template)

### Working Title
**"Mind the Gap: Why Lakehouse Interoperability Needs a Composable Index Substrate, Not Just Metadata Translation"**

Alternative: *"Beyond Format Conversion: Toward a Lakehouse Index Protocol for Composable Metadata"*

### Nugget (1 sentence)
Metadata translation tools like XTable and UniForm convert the 10% of lakehouse metadata any engine can already read, while the indexes and statistics that deliver 10-100x performance gains remain format-locked — what's missing is not conversion, but a standard index discovery and consumption protocol.

### Section-by-Section Outline

**Section 1: Introduction (0.75 pages)**
- The multi-engine lakehouse promise: swap engines freely
- Reality: engines see different subsets of the same table's metadata
- XTable/UniForm solve schema and file-listing translation but NOT index/stats translation
- Thesis: composable lakehouses need a standard index substrate with a discovery protocol
- Cite Pedreira et al. Composable DMS Manifesto (VLDB 2023) — metadata is the missing layer

**Section 2: What Gets Lost in Translation (1.25 pages)**
- **Cross-format index comparison table** (core empirical contribution):
  
  | Capability | Hudi MDT | Iceberg | Delta Lake | XTable translates? |
  |------------|----------|---------|------------|-------------------|
  | File listings | files partition | manifest list → manifests | _delta_log JSON/Parquet | Yes |
  | Column statistics | column_stats partition (HFile) | manifest per-file stats | checkpoint stats columns | Partial (schema differs) |
  | Partition stats | partition_stats partition | partition metadata | partition columns in log | Partial |
  | Bloom filters | bloom_filters partition | Puffin blob (apache-datasketches) | N/A | No |
  | Record-level index | record_index partition | N/A (Iceberg V3 row lineage proposed) | N/A | No |
  | Secondary index | secondary_index partition | N/A | N/A | No |
  | Expression index | expr_index partition | N/A | Generated columns + stats | No |
  | Concurrent backfill | 3-phase protocol | N/A | N/A | No |

- **Engine capability matrix**:
  
  | Engine | Hudi file listing | Hudi col stats | Hudi bloom | Hudi RLI | Hudi secondary | Iceberg stats | Delta stats |
  |--------|------------------|---------------|-----------|---------|---------------|--------------|------------|
  | Spark | Full | Full | Full | Full | Full | Full | Full |
  | Flink | Full | Full | Full | Full | Partial | Full | Partial |
  | Presto | Partial | No | No | No | No | Full | Partial |
  | Trino | Dropped (419+) | — | — | — | — | Full | Full |
  | Athena | File listing only | No | No | No | No | Full | Full |
  | DuckDB | No (community) | No | No | No | No | Full | Partial |

- Quantify the "interoperability tax": cite FORMATS paper numbers — secondary index reduces data scanned by 99% (759GB → 593MB), RLI 64x faster than global simple at 36B rows. Engines without index access pay this full cost.

**Section 3: Hudi's Metadata-as-Data as a Composable Substrate (1 page)**
- Brief architecture overview (reuse/cite FORMATS paper — complementary deep-dive)
- Frame through composability lens (NEW content):
  - **Modality composability**: partition-per-index design — adding new index requires only defining a new partition, no protocol/storage changes
  - **Lifecycle composability**: table services (compaction, clustering, indexing) are independently deployable — inline, async, or fully decoupled
  - **Engine composability**: connector layer abstracts index access (but with inconsistent capabilities, per Section 2)
- Contrast briefly with Delta (monolithic log — stats tied to checkpoint format) and Iceberg (manifest hierarchy + Puffin sidecars — cleaner but still format-coupled)

**Section 4: The Lakehouse Index Protocol (LIP) (1.5 pages)**
- **Design principles** drawn from LSP and CIFF (Common Index File Format from IR):
  1. Engines should discover indexes without format-specific code
  2. Capability negotiation: engine declares what it can consume, table advertises what's available
  3. Index data exchange in a format-neutral wire format
  
- **Protocol specification** (3 operations):
  ```
  discover(table_uri) → IndexManifest
    // Returns: list of {index_type, schema, freshness_guarantee, access_pattern}
  
  negotiate(engine_capabilities, index_manifest) → UsableIndexSet
    // Returns: subset of indexes the engine can safely consume
    // Handles: version compatibility, partial reads, fallback strategies
  
  read(index_id, key_range, read_options) → IndexData
    // Returns: index entries in a format-neutral encoding
    // Supports: point lookups, prefix scans, range scans
  ```

- **Capability manifest schema** (embedded in table metadata):
  ```json
  {
    "format_version": 1,
    "indexes": [
      {
        "type": "column_stats",
        "columns": ["ws_ship_customer_sk", ...],
        "storage_format": "hfile",
        "freshness": "transactionally_coupled",
        "access_patterns": ["point_lookup", "prefix_scan"],
        "schema_uri": "lip://schemas/column_stats/v1"
      }
    ]
  }
  ```

- **Worked example**: Trino discovering and consuming a Hudi secondary index via LIP, without implementing Hudi-specific code. Show the 3-step flow with a sequence diagram.

- **Cross-format applicability**: show how Iceberg Puffin blobs and Delta checkpoint stats could also be exposed through LIP.

**Section 5: Discussion (1 page)**
- **Lessons from LSP adoption**: LSP succeeded because (a) Microsoft shipped a reference implementation, (b) the protocol was simple enough for weekend implementations, (c) it solved a real pain point. LIP should aim for the same simplicity.
- **Self-describing index partitions**: propose that table formats embed a machine-readable capability manifest (like OpenAPI/WSDL for web services). Hudi could implement this as a `_capabilities` partition in the metadata table.
- **Limitations**: LIP does not address write-side index maintenance (engines would still need format-specific writers). This is deliberate — read-side interoperability is the tractable first step.
- **Relation to DuckLake**: DuckLake externalizes metadata to a relational database, which could serve as a LIP server. This shows LIP is compatible with all three metadata architecture families.
- **Connection to Composable DMS Manifesto**: position LIP as the missing piece in the Ibis/Substrait/Calcite/Velox stack.

**Section 6: Related Work and Conclusion (0.5 pages)**
- Composable Data Management Manifesto (Pedreira et al., VLDB 2023)
- Active Data Lakes (TUM, 2026) — physical data independence
- DuckLake (Raasveldt & Muhleisen, 2025)
- XTable / UniForm
- Shared Foundations at Meta (CIDR 2023)
- CIFF from IR community
- LSP from developer tools
- Call to action: the lakehouse community should invest in index interoperability, not just format conversion

### Key References to Add (beyond FORMATS paper)
1. Pedreira et al. "Composable Data Management System Manifesto" VLDB 2023
2. TUM "Active Data Lakes" 2026
3. DuckLake announcement/paper 2025
4. Iceberg Puffin specification
5. Delta Lake UniForm documentation
6. Apache XTable documentation
7. LSP specification (Microsoft)
8. CIFF paper (Lin et al., SIGIR 2020)
9. Meta "Shared Foundations" CIDR 2023
10. Jack Vanlightly "Beyond Indexes" analysis 2025

---

## Distinctness from FORMATS Paper

| Aspect | FORMATS '26 | CDMS '26 (this paper) |
|--------|-------------|----------------------|
| Focus | Hudi indexing internals | Cross-format composability + protocol |
| Scope | Single system (Hudi) | Three formats + protocol proposal |
| Contribution | Architecture + evaluation | Interoperability gap analysis + LIP |
| Audience | Table format designers | Composable systems community |
| Reuse | — | ~1 page of Hudi architecture (Section 3) |
| New content | — | Cross-format tables, engine matrix, LIP spec, LSP/CIFF connections |

---

## Critical Files

| File | Role |
|------|------|
| `main.tex` | FORMATS submission — reference for what's already published. Cite but don't repeat. |
| `hudi_formats26_draft.tex` | Earlier draft with interoperability section (Section 5) — starting material for Section 2. |
| `Metadata_as_data.pdf` | 11-section outline — background on table services and extensibility. |
| **New file: `cdms_paper.tex`** | The CDMS paper — create new, VLDB workshop template. |

---

## Writing Plan & Timeline

### Phase 1: Research & Tables (Week 1-2)
1. Build the **cross-format index comparison table** (Section 2) — systematically verify each cell by reading Iceberg Puffin spec, Delta Lake docs, and XTable source
2. Build the **engine capability matrix** (Section 2) — verify each engine's actual index support level from connector docs/source code
3. Read the TUM "Active Data Lakes" paper carefully — understand overlap and differentiation
4. Read the Composable DMS Manifesto — extract the composability dimensions to use as framework

### Phase 2: Protocol Design (Week 2-3)
1. Design the **LIP (Lakehouse Index Protocol)** specification — 3 operations, capability manifest schema
2. Create the **worked example** — Trino discovering a Hudi secondary index via LIP
3. Draft the **sequence diagram** for LIP flow
4. Sketch how Iceberg Puffin and Delta stats would be exposed through LIP

### Phase 3: Writing (Week 3-5)
1. Set up VLDB workshop LaTeX template
2. Write Section 1 (Introduction) — the "beyond format conversion" hook
3. Write Section 2 (What Gets Lost) — tables + quantified interoperability tax
4. Write Section 3 (Hudi as Composable Substrate) — reuse + composability framing
5. Write Section 4 (LIP) — protocol spec, schema, worked example
6. Write Section 5 (Discussion) — LSP lessons, DuckLake connection, limitations
7. Write Section 6 (Related Work + Conclusion)

### Phase 4: Review & Polish (Week 5-6)
1. Internal review with co-authors (Prashant, Sivabalan)
2. Ensure distinctness from FORMATS paper
3. Verify all cross-format claims against current specs
4. Polish figures and tables for 6-page fit

---

## Verification

1. **Compile check**: `latexmk -pdf cdms_paper.tex` with VLDB workshop template
2. **Distinctness check**: Side-by-side comparison with FORMATS paper — no paragraph-level overlap beyond the ~1 page architecture overview
3. **Cross-format accuracy**: Every cell in the comparison tables verified against official documentation (Iceberg spec, Delta protocol, Hudi tech spec)
4. **Protocol completeness**: LIP spec covers discover/negotiate/read with concrete schemas and at least one worked example
5. **CDMS topic alignment**: Paper addresses at least 3 of the 2025 CFP topics: (a) interoperability across systems, (b) exchange protocols and APIs, (c) open source composable modules

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| "Hudi marketing paper" perception | Lead with the cross-format problem, not Hudi. Hudi is one case study. Show how LIP benefits Iceberg/Delta too. |
| XTable/Onehouse conflict of interest | Frame as "we built the conversion tool, here's what it can't do" — credible self-critique |
| Protocol too vague | Include concrete JSON schemas, pseudocode, and a worked example with sequence diagram |
| CDMS 2026 doesn't happen | Backup: ADMS 2026 (deadline May 25) — reframe around storage architecture angle |
| Overlap with FORMATS paper | Strict budget: max 1 page of Hudi architecture reuse. Everything else is new. |

---

## Key Cross-Field Connections

- **LSP (Language Server Protocol)** → Lakehouse Index Protocol: M editors × N languages = M×N integrations, solved by standard protocol. Same for M engines × N index types.
- **CIFF (Common Index File Format from IR)** → Lakehouse Common Index Format: IR community solved incompatible inverted index formats across search engines.
- **CQRS / Event Sourcing** → Hudi's timeline as event bus for decoupled table services.
- **OpenAPI/WSDL** → Self-describing capability manifests for index partitions.

## Research Strategist Notes

- **Competitive window:** 6-12 months before a systems group (TUM, Databricks research, or cloud vendor) publishes something similar
- **Key researchers to watch:** Jana Giceva and Thomas Neumann (TUM), Raghu Ramakrishnan (Microsoft/Fabric), Ryan Blue (Tabular/Iceberg)
- **Monitor:** cdmsworkshop.github.io for CFP announcement; search "lakehouse index interoperability", "cross-format index protocol"
