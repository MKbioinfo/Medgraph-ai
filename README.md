# pfe-projet-2026

# Cardiology Knowledge Graph — SNOMED CT Extraction & Cleaning Pipeline

Part of the **AIMIG** (Assisted Intelligence in Medicine using Interactive Graphs) project — PFE (Final Year Project), Université Mohammed V - Rabat, Faculté de Médecine et de Pharmacie.

This repository contains the Google Colab notebook used to extract a cardiology-scoped subset of SNOMED CT and clean it into CSV files ready for import into Neo4j. It is the data engineering layer behind a cardiology clinical decision support knowledge graph.

## What this notebook does

Starting from the full SNOMED CT International Edition, the notebook extracts everything relevant to cardiology and outputs a small set of clean CSV files that can be loaded directly into Neo4j.

The extraction is rooted at the SNOMED CT concept **"Disorder of cardiovascular system" (SCTID 49601007)** and traverses the `IS_A` hierarchy (`typeId 116680003`) via breadth-first search to collect all descendant concepts.

The pipeline was built iteratively across four versions, each adding one capability on top of the last:

| Version | What it adds | Concepts | Descriptions | Relationships |
|---|---|---|---|---|
| **V1** | Minimal baseline: BFS over `IS_A` from the cardiology root, relationships kept only when *both* endpoints are in the cardiology set | 7,818 | 21,740 | 14,166 |
| **V2** | Relational enrichment: relationships kept when the *source* is in the cardiology set, regardless of destination; destination concepts outside the set are imported as `External`-type nodes | 11,629 | 33,611 | 47,461 |
| **V3** | Domain audit: traces every external concept up to one of SNOMED CT's top-level domain roots and applies a traced exclusion configuration (`EXCLUDED_ROOTS`, `EXCLUDED_SUBTREES`) to remove clinically irrelevant contamination (e.g. dermatology concepts reachable through a technically valid but clinically irrelevant IS-A path) | unchanged from V2 | unchanged from V2 | unchanged from V2 |
| **V4** | Final production graph: V3 exclusions applied, CSV files exported | **9,557** | **27,265 terms** | **40,650** |

V1 showed a structural limitation of the bilateral filter: about 96% of retained relationships were `IS_A`, since most non-hierarchical SNOMED relationships connect a cardiology concept to an external concept (e.g. an anatomical structure). V2 fixed this by relaxing the filter to the source side only, which raised relationship richness substantially and shifted the type distribution to a mixture (`FINDING_SITE`, `ASSOCIATED_MORPHOLOGY`, `OCCURRENCE`, `PATHOLOGICAL_PROCESS`, and others), but introduced cross-specialty contamination that V3's domain audit then removed.

## Why not just import all of SNOMED CT

SNOMED CT International Edition contains roughly 370,000 active concepts across every medical specialty. Filtering to cardiology before import, rather than importing everything and filtering afterward, keeps the resulting graph small and fast to query in Neo4j.

## Data source

- **SNOMED CT International Edition**, March 2026 production release
- Format: **RF2** (Release Format 2), tab-separated text files
- Files used: `sct2_Concept_Snapshot`, `sct2_Description_Snapshot-en`, `sct2_Relationship_Snapshot` (from the `Snapshot/Terminology` directory)
- Access: SNOMED CT is **not** freely licensed in Morocco (not a SNOMED International member country). Access was obtained via a UMLS Metathesaurus account through the U.S. National Library of Medicine.

**The raw SNOMED CT source files are not included in this repository** due to licensing terms. To run the notebook, you need your own UMLS-licensed copy of the SNOMED CT International Edition RF2 release.

## Output files

The notebook produces four CSV files, ready for `LOAD CSV` import into Neo4j:

| File | Contents |
|---|---|
| `nodes_concepts.csv` | `sctid`, `definitionStatus`, `conceptType` (`Cardiology` / `External`) |
| `nodes_terms.csv` | `termId`, `term` |
| `rels_has_term.csv` | `startId`, `endId`, `termType` (concept → term links) |
| `rels_snomed.csv` | `startId`, `endId`, `relType` (SNOMED semantic relationships) |

## Environment

- Google Colab + Google Drive
- Python (pandas)
- `csv.field_size_limit(2147483647)` — required because some SNOMED CT description fields exceed Python's default CSV field size limit
- Target database: Neo4j Desktop 2.1.4 with the APOC plugin (APOC is required downstream for `apoc.create.relationship()`, which creates dynamically typed relationships from the `relType` column)

## Data quality notes

- Both raw source files (Concepts, Descriptions) were confirmed duplicate-free; a duplication issue that appeared during development was traced to the processing pipeline and fixed with `drop_duplicates()` before joins.
- The V3 exclusion configuration follows two working rules: exclude branches, not individual leaf concepts, and verify a subtree's contents with a safety-check query before excluding it, rather than excluding based on a concept's name alone.
- Respiratory-system and social-context concepts were deliberately **kept** despite sitting outside a strictly cardiovascular branch, since conditions like pulmonary hypertension and risk factors like smoking are clinically relevant to cardiology.

## Relationship to the rest of the project

This extraction pipeline produces the **Phase 1 (ontological)** layer of the cardiology knowledge graph described in the accompanying thesis. A second phase enriches this graph with clinical reasoning relationships drawn from ESC (European Society of Cardiology) guideline content; that enrichment is applied directly in Neo4j and is not part of this notebook.

## License

No license has been specified for this repository yet. SNOMED CT itself remains subject to its own licensing terms (UMLS Metathesaurus / SNOMED International) and is not redistributed here.
