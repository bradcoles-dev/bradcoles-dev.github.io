# LinkedIn — delta-doctor launch post

---

Every Fabric Lakehouse I've looked at has the same problem: small files, accumulating deletion vectors, and no automated maintenance.

Dataflow Gen2, Copy activity, and Python kernel notebooks don't trigger Auto-Compaction. Spark notebooks have it but target 128 MB files, not the 256-400 MB that Silver and Gold layers need. Deletion vectors pile up silently after every MERGE and DELETE. On a neglected Lakehouse the fix is usually a VACUUM and an OPTIMIZE, not a bigger SKU.

Today I'm releasing delta-doctor: seven PySpark notebooks for automated Delta table maintenance on Fabric Lakehouses. Free, Apache 2.0, import directly into your workspace.

What's in it:
- A health scanner that classifies every table in seconds (file counts, avg file size, deletion vectors, clustering state) with no data scan
- Single-table and Lakehouse-wide OPTIMIZE + VACUUM, with OPTIMIZE gated on actual table health so it costs almost nothing when tables are already healthy
- A one-off rebaseline orchestrator for purging deletion vectors and right-sizing files on a Lakehouse that's never had maintenance applied
- Session config and table property notebooks to set the right defaults once

Built with hard constraints: VACUUM never runs below 7-day retention (enforced in code, not documentation), Gold-layer VACUUM requires explicit Direct Lake confirmation, and table enumeration works correctly for schema-enabled Lakehouses.

If you've been putting off Lakehouse maintenance because there was nothing to run, this is the starting point.

What does your current Delta maintenance look like?

#MicrosoftFabric #DeltaLake #DataEngineering #OpenSource

---

First comment:
Full write-up: https://bradcoles.dev/blog/introducing-delta-doctor.html
Docs and download: https://deltadoctor.dev
GitHub: https://github.com/bradcoles-dev/delta-doctor
