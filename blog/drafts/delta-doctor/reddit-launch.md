# Reddit — delta-doctor launch post

**Subreddit:** r/MicrosoftFabric

**Title:** I open-sourced a Delta table maintenance library for Fabric Lakehouses — 7 notebooks, free, Apache 2.0

---

I wrote a post a few months back on Delta table maintenance in Fabric. The follow-up I kept getting was "where's the code?" — so I built it out properly and released it.

**delta-doctor** is seven PySpark notebooks you import directly into your Fabric workspace via Import notebook. No package manager, no infrastructure.

**The notebooks:**

- `doctor_diagnosis_table_health`: health report across all tables in a Lakehouse: file counts, avg file size, fragmentation status, deletion vectors, liquid clustering state. Classifies each table. Runs in seconds; all metadata, no data scan.
- `doctor_treatment_table_maintenance`: OPTIMIZE + VACUUM on a single table, designed to sit at the end of a pipeline
- `doctor_treatment_maintenance_orchestrator`: same but across an entire Lakehouse; OPTIMIZE is gated so healthy tables are skipped automatically
- `doctor_treatment_rebaseline_orchestrator`: REORG TABLE APPLY (PURGE) + OPTIMIZE across the whole Lakehouse, for purging accumulated deletion vectors and right-sizing files on a Lakehouse that has never been maintained
- `doctor_prevention_session_config`: Spark session baseline by layer (Bronze/Silver/Gold)
- `doctor_prevention_set_table_properties`: sets Delta table properties per layer: deletion vectors, auto-compaction, optimize write, V-Order, target file size, liquid clustering
- `doctor_prevention_set_properties_orchestrator`: applies table properties to an entire Lakehouse in one run

**A few things I was deliberate about:**

- OPTIMIZE is gated on a `DESCRIBE DETAIL` check: if avg file size is within 80% of the layer target, the table is skipped. No unnecessary compute.
- VACUUM has a hard 168-hour floor enforced in code (`max(retain_hours, 168)`), not just documented
- Gold-layer VACUUM raises a `ValueError` if `direct_lake_confirmed` is not set to `True`; forces an explicit confirmation that your Direct Lake semantic model has re-framed to the latest Delta commit before VACUUM removes old files
- Table enumeration uses `mssparkutils.fs.ls()` + `_delta_log` detection, not `SHOW TABLES`, so schema-enabled Lakehouses work correctly
- Workspace GUID comes from `spark.conf.get("trident.workspace.id")` — not a parameter, not hardcoded

It's v0.1: the maintenance and health scanning core. The plan for v0.2 is health history logging and a Power BI dashboard built on that history, but these are directions not commitments.

Repo: https://github.com/bradcoles-dev/delta-doctor

Docs: https://deltadoctor.dev

Happy to answer questions. If you've hit edge cases with schema-enabled Lakehouses or liquid clustering in particular I'd be interested — those were the trickiest parts to get right.
