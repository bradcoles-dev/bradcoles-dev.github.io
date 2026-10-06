# Delta Table Maintenance in Microsoft Fabric: The Runtime 2.0 Edition

### Delta table maintenance in Microsoft Fabric after Runtime 2.0 GA: what's now automatic, what's still your job, and what I found when I tested it

**By Brad Coles | Associate Director - Data Engineering Capability Lead ANZ, Synechron Australia**

**Published 6 October 2026**

---

> **Accuracy notice:** This post was written in October 2026 and reflects the state of Microsoft Fabric Runtime 2.0 at that time. Fabric is a rapidly evolving platform - defaults may change, documentation may be corrected, and documented limitations may be resolved. Before making decisions based on this post, verify current behaviour against the [official Microsoft documentation](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization) and test it in your own environment. Where this post identifies gaps, contradictions or limitations, check whether they have been addressed in a recent Fabric release.

Earlier this year I published [Delta Table Maintenance in Microsoft Fabric: A 2026 Practitioner's Guide](https://bradcoles.dev/blog/fabric-delta-table-maintenance.html), written for Runtime 1.3, the current runtime at the time, with notes on the Runtime 2.0 Preview. I wrote it because Fabric is sold as SaaS, and most teams reasonably assume that table maintenance is taken care of for them. Unfortunately it isn't, and at the time the official documentation was thin, scattered and inconsistent.

[Runtime 2.0](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0) went GA on 14 August 2026, and it's now the default runtime for new workspaces. Much of my old advice is now defaults - I'd like to take credit for the good work of Miles Cole and Fabric's Spark team, though I know better, at least it's a good endorsement of my previous article if nothing else! Adaptive file sizing, Fast Optimize, deletion vectors and incremental liquid clustering are all on by default now. The big ticket improvement, incremental liquid clustering, removes the full-table rewrite that made liquid clustering impractical on Runtime 1.3.

For better or worse, in Runtime 2.0 **Fabric still doesn't maintain your Delta tables for you.** OPTIMIZE doesn't run on a schedule - auto-compact makes up some ground here but is disabled by default - and there's no auto-VACUUM.

If you want to manage your costs and capacity usage effectively, there's still manual tuning required. Microsoft [announced on-demand billing and a zero-provisioned F0 SKU][f0] at FabCon Europe in September, which should give spiky workloads a middle ground, but pricing isn't published yet, and on-demand billing doesn't make wasted compute free. Poor maintenance inflates capacity usage gradually, small files multiply, deletion vectors pile up, Direct Lake falls back to cold-state transcoding. That creeping usage is hard to diagnose, often teams assume they're just processing more data, it's easy to blame on organic growth. But eventually you'll outgrow your SKU and be forced to upgrade when the real fix might have been reading this article and implementing a few lines of configuration.

My research and testing of Runtime 2.0 revealed this is actually a bigger problem under Runtime 2.0. Deletion Vectors are now enabled by default, so every UPDATE, DELETE and MERGE means they start to pile up.

For this edition I trawled through the documentation and ran a notebook against a Runtime 2.0 workspace (Spark 4.1.1, Delta 4.2.0) and a Runtime 1.3 environment. I checked every default this guide relies on, benchmarked liquid clustering on both runtimes, and tested how Data Factory's Copy activity and Copy job behave against Runtime 2.0 tables. My work revealed many documentation and runtime contradictions, a reminder to trust your environment over any documentation, including this article!

> **What's changed since my original guide.** Some of its advice is now redundant because Runtime 2.0 made it the default. Some was overtaken by Microsoft's revised guidance. And a couple of points were wrong, or are now wrong on Runtime 2.0:
> - **The 256 MB Silver/400 MB Gold file-size targets** (and 400 MB–1 GB with 8M-row row groups for Direct Lake). These came from Microsoft's own [Cross-Workload page](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization), which [dropped fixed targets on 26 August](https://github.com/MicrosoftDocs/fabric-docs/commit/2e2cc1c5e8c6f536d95e0066da132ab46380cdbf) in favour of adaptive file sizing, and now gives Direct Lake a 1–16 million row-group range.
> - **"Run OPTIMIZE aggressively at Silver and Gold."** Also from that page, and also removed. Microsoft's default strategy is now Auto Compaction, with scheduled OPTIMIZE kept for specific cases.
> - **The V-Order figures (40–60%/~10%/15–33%) and enabling it for SQL analytics endpoint consumers.** The 40–60% figure was on the same page and went in the same revision. Microsoft now says to enable V-Order only when Direct Lake is a primary consumer.
> - **"Optimize Write is on by default."** Under the default `writeHeavy` profile it only applies to partitioned tables.
> - **"Fast Optimize does not apply to liquid clustering."** That's what the [Table Compaction page](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#fast-optimize) said. On Runtime 2.0 it does apply, and I measured it.
> - **No liquid clustering at Bronze.** Bronze MERGE targets benefit as much as any table.
>
> **Scope:** Lakehouse (Spark/Delta) tables only. Fabric Warehouse manages its own layout and is out of scope.

---

## Runtime 2.0 at a Glance

If you only read one section, read this one.

| Setting | Runtime 1.3 | Runtime 2.0 | What you do now |
|---|---|---|---|
| Adaptive Target File Size (ATFS) | Opt-in | **On** | Nothing. Remove it from your utility notebook. |
| File-Level Compaction Target | Opt-in | **On** | Nothing. |
| Fast Optimize | Opt-in | **On** | Nothing, but know when it skips work (see [Compaction](#compaction)). |
| Deletion vectors | Opt-in | **On for new tables** | Keep them. Schedule OPTIMIZE on update-heavy MERGE targets. |
| Incremental liquid clustering | Not available (full rewrite under 100 GB) | **On** | Use liquid clustering by default. |
| Auto Compaction | Off | **Off** | **Enable it**, preferably as a table property. |
| V-Order | Off | Off | Enable for Direct Lake tables, or use `readHeavyForPBI`. |
| Optimize Write | Partitioned tables only (`writeHeavy`) | Same | Enable per table for streaming/microbatch writes. |
| Native Execution Engine | Opt-in | Opt-in (now also accelerates liquid clustering OPTIMIZE) | Enable it. |
| VACUUM LITE, `OPTIMIZE FULL`, `clusteringQuality()` | Not available | Available | Use where relevant. |

Every "On" in the Runtime 2.0 column is something I confirmed in a fresh session, both from the session config and by testing the behaviour. You can check your own environment with this snippet:

```python
for k in ["spark.microsoft.delta.optimize.fast.enabled",
          "spark.microsoft.delta.optimize.fileLevelTarget.enabled",
          "spark.microsoft.delta.targetFileSize.adaptive.enabled",
          "spark.databricks.delta.autoCompact.enabled",
          "spark.databricks.delta.optimizeWrite.enabled",
          "spark.sql.parquet.vorder.default",
          "spark.microsoft.delta.optimize.clustering.strategy.incremental",
          "spark.databricks.delta.properties.defaults.enableDeletionVectors"]:
    try:
        print(f"{k:70} {spark.conf.get(k)}")
    except Exception:
        print(f"{k:70} <unset>")
```

My original guide was largely a list of configs to set, Runtime 2.0 now sets most of them for you. Miles Cole's own summary at the [August Fabric Spark AMA][ama] was that incremental liquid clustering, ATFS, Fast Optimize and the file-level target are on by default with "no reason to disable" them, and that his only two non-default go-tos are the Native Execution Engine and Auto Compaction. All that's left is a short list of decisions that you'll need to make on your own, dependent on your environment and workloads, and the rest of this guide is about those.

---

## Fabric Still Doesn't Maintain Your Tables

**What happens automatically:**

- **Delta log checkpointing** every 10 commits.
- **File statistics** (min, max, null counts) on every write, which power file skipping.
- **Write-time defaults:** adaptive file sizing, deletion vectors on new tables, and Optimize Write on partitioned tables.

**What doesn't:**

- **Scheduled OPTIMIZE.** Fabric never schedules OPTIMIZE for you. (Auto Compaction is different: it runs OPTIMIZE automatically after a write, but only once you enable it. More on that below.)
- **VACUUM.** Fabric never vacuums your unreferenced files, these stay in OneLake and you pay for them.
- **Auto Compaction.** It's still off by default, even though Microsoft's [cross-workload guidance](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization) now says to *"enable auto compaction as the default maintenance strategy."*

The Lakehouse explorer has a right-click **Maintenance** action for one-off OPTIMIZE and VACUUM on a single table. Pipelines now have a [**Lakehouse Maintenance activity**](https://learn.microsoft.com/en-us/fabric/data-factory/lakehouse-maintenance-activity) that [went GA in Sep 2026][f0] despite being painful to use, and incompatible with schema-enabled Lakehouses. I didn't get as far as testing whether it clusters liquid clustered tables since I exclusively use schema-enabled Lakehouses, and had already had a gutful trying to get the activity to work (I may also be impatient).

The right maintenance strategy varies by table, e.g. an append-only Bronze table and a Gold table serving hundreds of Direct Lake users need different things, and a single automated policy would either waste capacity on the first or under-serve the second. That said, Runtime 2.0 has narrowed the gap with much better defaults.

Personally, I'd still like an opt-in automatic mode for teams that want one (or can't be bothered tuning their own and are happy with 'good enough'). Databricks has [predictive optimisation](https://docs.databricks.com/aws/en/optimizations/predictive-optimization), which decides per table when to run OPTIMIZE and VACUUM. Fabric has nothing equivalent, and Cole said at the [AMA][ama] there are no plans for a `CLUSTER BY AUTO`.

---

## Compaction

### How it works

Most practitioners would (should) know that every write creates new Parquet files. Frequent small writes, MERGEs and updates leave behind lots of small files, and every query pays to open, read the footer of, and skip each one. Compaction is the panacea to the small-file problem, it rewrites many small files into fewer, right-sized ones.

There are two ways to compact in Fabric:

- **OPTIMIZE**: a command you run (or schedule).
- **Auto Compaction**: an OPTIMIZE that runs in the same session immediately after a write, when it detects too many small files. In my testing it fired at exactly 50 small files, which matches the [documented `minNumFiles` default](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#tune-auto-compaction-thresholds). That threshold is tunable (`spark.databricks.delta.autoCompact.minNumFiles`), but I'd leave it - lowering it makes compaction fire more often in your writes and it doesn't help with the deletion-vector case below, which isn't about small files at all.

**Fast Optimize** sits in front of OPTIMIZE. Before rewriting a bin of files, it checks whether doing so would meaningfully improve the table, and skips the bin if not. That makes a no-op OPTIMIZE nearly free.

### What changed in Runtime 2.0

- **Fast Optimize is on by default.** I couldn't find any documentation that notes this, and the [Table Compaction page](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#fast-optimize) still shows it as something you set. But my testing showed it's on by default in RT 2.0 - running OPTIMIZE on a table with 10 tiny files made **no commit at all**, but with Fast Optimize disabled the same command compacted 10 files into 1.
- **Fast Optimize applies to liquid clustered tables too.** Bear with me on this: my original guide said it didn't, following the [Table Compaction page](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#fast-optimize), which still says it's *"not applicable to liquid clustering"*, yet the [Liquid Clustering page](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#interaction-with-other-features) says it's *"compatible starting in Runtime 2.0"*. This rattled me so I tested it: I ran ten tiny appends to a liquid clustered table, these were skipped by a default OPTIMIZE and clustered once Fast Optimize was disabled.
- **Auto Compaction is now Microsoft's recommended default strategy.** That's a significant shift. Until 26 August, Microsoft's [Cross-Workload page](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization) said to run scheduled OPTIMIZE *"aggressively"* at Silver and Gold, which is where my original guide got it. The revision that removed it is titled, in the [docs' public change history](https://github.com/MicrosoftDocs/fabric-docs/commit/2e2cc1c5e8c6f536d95e0066da132ab46380cdbf), *"Simplify data layout guidance based on benchmarking"*.
- **Auto Compaction also clusters on liquid clustered tables.** I confirmed this: OPTIMIZE ran with `auto = true`, `clusterBy` set, and the new files carried `clusteringProvider = liquid`. This is a big tick for Auto Compaction in my book, by handling small files and clustering it lessens the reliance on a scheduled OPTIMIZE.
- [**`onCheckpointOnly`**](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#reduce-evaluation-overhead) (`spark.microsoft.delta.autoCompact.onCheckpointOnly.enabled`) defers Auto Compaction's evaluation to checkpoints, which happen every 10 commits. The snapshot is already in memory at that point, so the check is close to free. This is worth enabling for high-commit-rate pipelines.

### What to do

**Enable Auto Compaction, and do it as a table property.** Session configs only apply to the session that sets them, so any writer that doesn't run your utility notebook won't trigger compaction. A table property applies to every writer. In my testing the table property won even when the session config was explicitly `false`.

```sql
ALTER TABLE silver.my_table
SET TBLPROPERTIES ('delta.autoOptimize.autoCompact' = 'true');
```

**Know what Fast Optimize skips.** It deliberately ignores a handful of small files. That's the point of it, but it means a trickle of tiny appends can sit uncompacted (and, on liquid clustered tables, unclustered) for a while. If you need to force a compaction, turn it off for that one command:

```python
spark.conf.set("spark.microsoft.delta.optimize.fast.enabled", "false")
spark.sql("OPTIMIZE silver.my_table")
spark.conf.unset("spark.microsoft.delta.optimize.fast.enabled")
```

**Keep a scheduled OPTIMIZE only where it earns its place:**

1. **One-off backlog cleanup** on tables that have never been maintained.
2. **Update-heavy MERGE targets.** Auto Compaction doesn't clear deletion vectors unless small files also trigger it (see [Deletion Vectors](#deletion-vectors)). This is now the main reason scheduled OPTIMIZE is still required, particularly for tables serving Direct Lake Semantic Models.
3. **Workloads that can't tolerate write latency.** Auto Compaction runs in the write path, so for these workloads you'd disable (or not enable) Auto Compaction, and rely on a scheduled OPTIMIZE for compaction and clustering.
4. **Direct Lake tables, timed around framing.** Direct Lake models pick up new data by [*framing*](https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-how-it-works#framing) the latest Delta version. With [automatic updates](https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-how-it-works#automatic-updates) on (the default), that happens whenever the table changes, so run OPTIMIZE straight after the load that created the deletion vectors. If you've turned automatic updates off to control when data appears, finish OPTIMIZE before your scheduled refresh.

Use `OPTIMIZE ... DRY RUN` to see what a run would rewrite before committing to it. Auto Compaction runs appear in `DESCRIBE HISTORY` as `OPTIMIZE` with `auto = true`, along with file counts and size percentiles, which is the easiest way to confirm it's working.

---

## File Sizing

### How it works

Every compaction, every OPTIMIZE and every Optimize Write aims at a target file size. Too small and you're back to the small-file problem. Too large and you lose parallelism and skipping granularity.

[**Adaptive Target File Size (ATFS)**](https://learn.microsoft.com/en-us/fabric/data-engineering/tune-file-size#adaptive-target-file-size) sets that target from the table's size: a flat 128 MB for any table under 10 GB, scaling linearly to 1 GB at 10 TB. 1 GB is the ceiling: it's the default `maxFileSize` (configurable between 128 MB and 1 GB), and once a table reaches it ATFS stops re-evaluating (`stopAtMaxSize`). The evaluated value is stored on the table as the `delta.targetFileSize.adaptive` property, so you can read it from `DESCRIBE DETAIL`.

Is 1 GB too small for a very large table? I don't think so. Bigger files mean fewer files to list, but every MERGE, UPDATE or deletion-vector purge has to rewrite whole files, so larger files make those more expensive, and fewer files give file skipping less to work with. Microsoft caps it at 1 GB and doesn't let you go higher.

The [**File-Level Compaction Target**](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#file-level-compaction-targets) tags each compacted file with the target it was written for. When ATFS raises the target as the table grows, files that were at least half the target at the time they were compacted aren't rewritten (e.g. a 100 MB file compacted under a 128 MB target is left alone when the target rises to 256 MB, even though it's well under the new target, whereas a 50 MB file would get compacted). That avoids needless rewrites as tables grow.

These are both **runtime defaults, not table settings**, so they apply to existing tables as soon as a Runtime 2.0 session touches them. ATFS evaluates a table's target at the start of its next OPTIMIZE (or on CTAS and overwrites), and existing files aren't rewritten just because the setting is now on. In my migration test, the files Runtime 1.3 had written were left alone. The one thing to check is that no Environment or notebook still sets these configs to `false`.

Christopher Finlan (Microsoft) [described the "compaction gap" well](https://christopherfinlan.com/2026/02/15/microsoft-fabric-table-maintenance-optimization-a-cross-workload-survival-guide/): without ATFS, Auto Compaction aims at 128 MB while OPTIMIZE aims at 1 GB, so the two never converge and you end up recompacting forever. With ATFS they share one dynamic target, which is another reason why the scheduled OPTIMIZE has become less necessary.

### What changed in Runtime 2.0

- **ATFS and the File-Level Target are both on by default.** The [Table Compaction page](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#file-level-compaction-targets) still says the File-Level Target is *"not enabled by default"*. The [Tune File Size page](https://learn.microsoft.com/en-us/fabric/data-engineering/tune-file-size#understand-performance-impact) says it's *"enabled by default starting in Runtime 2.0"*. My testing confirmed the latter is right.
- **Fixed per-layer targets are gone from the documentation.** Until 26 August, the Cross-Workload page gave per-consumer targets ([see the change](https://github.com/MicrosoftDocs/fabric-docs/commit/2e2cc1c5e8c6f536d95e0066da132ab46380cdbf)): 400 MB for the SQL analytics endpoint, 400 MB–1 GB with 8-million-row row groups for Direct Lake, and 128–256 MB at Silver. That's where my 256 MB/400 MB targets came from. Microsoft's [current advice](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#cross-workload-guidance) is *"Don't set a static target file size, arbitrary row limit, or V-Order solely for SQL analytics endpoint performance,"* and `delta.targetFileSize` is now [described](https://learn.microsoft.com/en-us/fabric/data-engineering/tune-file-size#set-target-file-size-consistently) as existing *"for legacy compatibility purposes."*

### What to do

- **Leave ATFS alone.** If you want to know a table's target, read it rather than setting it:
  ```python
  spark.sql("DESCRIBE DETAIL silver.my_table").first()["properties"].get("delta.targetFileSize.adaptive")
  ```
- **Remove any `delta.targetFileSize` pins** unless you can justify them against a measured workload.
- **Use Optimize Write selectively.** The default `writeHeavy` profile leaves `optimizeWrite.enabled` unset and applies Optimize Write to partitioned tables only. Microsoft recommends it mainly for streaming and microbatch writes. Enable it per table rather than for everything:
  ```sql
  ALTER TABLE silver.my_streaming_table
  SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true');
  ```

---

## Clustering

This is where Runtime 2.0 changed the most, so it gets the most space.

### How it works

**Partitioning** physically splits a table into a folder per value of a column (e.g. one folder per `date`). Queries filtering on that column skip whole folders. The trouble is that it bakes a physical structure into the table: a high-cardinality column produces thousands of tiny partitions (the small-file problem again, at a structural level), and choosing the wrong column means rewriting the table. Microsoft's [cross-workload guidance](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#organize-data-for-file-skipping) now says *"Avoid partitioning by default."*

**Z-Order and liquid clustering** both rearrange rows so that similar values end up in the same files. Each file then covers a narrow range of values, and its min/max statistics let queries skip the files that can't contain what they're looking for. With ATFS, a 10 GB table is roughly 80 files of ~128 MB. Unclustered, a selective query on a given column may have to open all of them. Clustered on that column, it may only need one: 128 MB read instead of 10 GB. A table split into N files can skip at most (N−1)/N of them, so clustering only pays off once a table spans several files, and only as well as the clustering quality allows.

**Liquid clustering is the better of the two**, for four reasons:

- **The layout lives on the table.** You declare it once with `CLUSTER BY`, and every OPTIMIZE maintains it, including Auto Compaction. Z-Order is a one-off operation: you name the columns every time you run `OPTIMIZE ... ZORDER BY`, and nothing tells the next person which columns to use.
- **It's incremental.** On Runtime 2.0, OPTIMIZE only reclusters the files that need it (more on that below).
- **You can change keys without a rewrite.** `ALTER TABLE ... CLUSTER BY` changes the policy going forward, where switching Z-Order columns means re-running it across the table.
- **It handles multiple columns better.** For two or more cluster columns, Fabric uses a Hilbert curve, which keeps neighbouring values together more consistently than Z-Order's bit-interleaving. (For a single column both reduce to a sort, so the first three points are the win.)

**Clustering never happens on write.** A plain INSERT or MERGE writes unclustered files, even on Runtime 2.0. Miles Cole's [reasoning][ama] is that clustering on write forces a shuffle that can make writes twice as slow or worse. OPTIMIZE is a guarantee where cluster-on-write would only be optimistic. Clustering only happens when OPTIMIZE runs, either explicitly or through Auto Compaction (which, on a liquid clustered table, clusters as well as compacts). If neither ever runs, `CLUSTER BY` gives you no benefit at all. The same goes for Data Factory - a Copy activity or Copy job append into a liquid clustered table writes unclustered files.

### What changed in Runtime 2.0

**Runtime 1.3 rewrote your whole table on every OPTIMIZE.** Open-source Delta groups clustered files into Z-Cubes (a Z-Cube is a group of files clustered on the same columns, which keeps growing until it reaches 100 GB or the cluster keys change). Any Z-Cube under 100 GB is "partial" and is rewritten in full whenever there's new data to cluster, until it seals at 100 GB. Most tables are under 100 GB, so most tables were rewritten in full on every OPTIMIZE. Miles Cole, [on r/MicrosoftFabric][cole-reddit]: *"Databricks doesn't even work this way, it's really just the OSS LC implementation."*

Tables under 100 GB (most tables) fit inside one Z-Cube. Let's say you have a 50 GB table that has recently been compacted and clustered and you do a 100 MB write to this table, which adds a new file outside the Z-Cube. Under Runtime 1.3, when you run OPTIMIZE this entire table 50.1 GB is rewritten. Now with Incremental Liquid Clustering on Runtime 2.0, the existing Z-Cube is left alone, and only the 100 MB of new data (plus any partly filled file it merges with) is re-written into the Z-Cube.

**Runtime 2.0 [clusters incrementally](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#incremental-liquid-clustering).** A file is only rewritten if any of these are true:

- it's unclustered (e.g. new data)
- it's too small
- more than 5% of its rows are deleted via deletion vectors
- it's already clustered but overlaps too much with its neighbours (Auto Reclustering)

Everything else is left alone.

I couldn't find a published Fabric benchmark for the difference, so I ran one. I used a 20-million-row, ~1.45 GB table clustered on two columns, with settings pinned so that only the clustering algorithm differed: Fast Optimize, Optimize Write and Auto Compaction all off. After the initial OPTIMIZE, I ran three rounds of "append 1% new data, then OPTIMIZE":

| Round | Runtime 1.3 rewrote | Runtime 1.3 time | Runtime 2.0 rewrote | Runtime 2.0 time |
|---|---|---|---|---|
| 0 | 1.47 GB (101%) | 54 s | 15 MB (1.0%) | 6 s |
| 1 | 1.48 GB (102%) | 53 s | 30 MB (2.1%) | 6 s |
| 2 | 1.50 GB (103%) | 54 s | 44 MB (3.1%) | 8 s |

On Runtime 1.3, clustering about 15 MB of new data meant rewriting the entire table every time: roughly 100 times the new data, and 7–9 times slower. The gap only grows with table size, because Runtime 1.3's cost scales with the whole table and Runtime 2.0's scales with the new data. Note that this is just one table and one append pattern, not a universal multiplier.

Two details are worth noticing:

- **Incremental Clustering on Runtime 2.0 isn't quite "only the new data".** Each round rewrote the new data *plus* the previous round's partly-filled clustered file, which keeps growing until it reaches the target size.
- **Runtime 1.3 put the whole clustered table in a single 1.45 GB file**, where Runtime 2.0 wrote 10 files of about 145 MB. A single file can't be skipped, so on Runtime 1.3 a table this size paid for a full rewrite on every OPTIMIZE *and* got no file skipping in return.

There's a trade-off hidden in those numbers. In a separate test, each newly clustered file covered the full range of both cluster keys, so it overlapped every other file and no query could skip it. Incremental clustering keeps rewrites small by clustering new data only within itself, so a small slice of recent data stays unskippable until Auto Reclustering (or an OPTIMIZE FULL) reorganises it with its neighbours.

The guidance changed accordingly. In February, [Christopher Finlan wrote](https://christopherfinlan.com/2026/02/15/microsoft-fabric-table-maintenance-optimization-a-cross-workload-survival-guide/) that partitioning *"is often the better choice until Runtime 2.0."* In May, [Miles Cole][cole-reddit]: *"I fully recommend using Liquid Clustering over partitioning and Z-Order, as long as you are using Runtime 2.0."*

**Migration from 1.3 is automatic.** At the [AMA][ama], Cole's answer to "is there a migration step?" was *"No migration, it's automatic and seamless :)"*. It's not that I don't trust Miles, or Microsoft, but I tested it. I took the table Runtime 1.3 had clustered, switched to Runtime 2.0, and repeated the same three rounds. They rewrote 1.0%, 2.0% and 3.0%, and the 1.5 GB file from Runtime 1.3 was never touched. There was no one-off full rewrite and no `OPTIMIZE FULL` required.

**`OPTIMIZE FULL` is still useful, just not required.** Migration stops the full-rewrite cost, but data clustered on Runtime 1.3 keeps its old layout, in my case that single 1.5 GB file. One `OPTIMIZE FULL` on Runtime 2.0 rewrote it into 11 files of 110–162 MB, the same layout a native Runtime 2.0 table gets. That's a single full rewrite, worth paying once on large or heavily queried tables you clustered on 1.3.

Also new in Runtime 2.0:

- [**`clusteringQuality()`**](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#evaluate-clustering-quality) (Scala only) reports per-column clustering health. See [Assessing Your Tables](#assessing-your-tables).
- **The Native Execution Engine now accelerates liquid clustering OPTIMIZE**, which Microsoft [quotes](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#apply-clustering-with-optimize) as *"30–50% faster multi-dimensional clustering."*
- [**Complex-type cluster keys**](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#supported-column-types) (struct, array, map) behind `spark.microsoft.delta.clusteredTable.complexTypes.enabled`.

### What to do

**Use liquid clustering by default**, with one to four cluster keys. Four is [Delta's limit](https://docs.delta.io/delta-clustering/), not just guidance. In practice, one or two uncorrelated, selective columns that appear in your filters and joins. The order you list them in `CLUSTER BY` doesn't matter. Low-cardinality or correlated columns waste a slot.

Cluster keys also need file statistics, and by default Delta only collects statistics on the [**first 32 columns**](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-file-skipping#default-behavior) of a table. If a key sits beyond column 32, move it earlier or name it in `delta.dataSkippingStatsColumns`. (`clusteringQuality()` reports `no_stats` for a key without statistics.)

```sql
CREATE TABLE silver.orders (...) CLUSTER BY (customer_id);

-- Existing table: change the policy without rewriting existing data
ALTER TABLE silver.orders CLUSTER BY (customer_id, order_date);
```

**MERGE targets benefit, but not quite how I expected.** Cluster the target table on the merge key or, for a composite key, its most selective one or two columns, but never a hash of the key, which scatters neighbouring rows across every file.

It's often said that the MERGE `ON` clause works like a filter, so a clustered target lets MERGE skip files. I tested that and it's only half true.

A MERGE reads the target twice:

1. A *matching* scan to find which files contain the batch's keys, then
2. A *rewrite* scan of just those files.

I merged the same batch (50,000 updates to a single block of consecutive order IDs, plus 10,000 new orders) into two identical 1.4 GB tables, one clustered on `order_id` and one not:

| | Matching scan | Rewrite scan | Deletion vectors added |
|---|---|---|---|
| Plain `ON`, clustered | 18 files, 1,368 MiB | **1 file, 127 MiB** | **1** |
| Plain `ON`, unclustered | 18 files, 1,368 MiB | 10 files, 1,360 MiB | 10 |
| `ON` + key range, clustered | **9 files, 396 MiB** | **1 file, 132 MiB** | **1** |
| `ON` + key range, unclustered | 16 files, 1,362 MiB | 10 files, 1,360 MiB | 10 |

- **A plain `ON t.key = s.key` doesn't skip files.** The matching scan read every file on both tables. Fabric doesn't prune target files from join keys.
- **Clustering still pays off in the rewrite.** The changed rows sat in one file instead of ten, so the MERGE read about 45% less in total and left 1 deletion vector instead of 10. That's less for the next OPTIMIZE to purge and less for Direct Lake to load.
- **Add a range on the target key and clustering skips files too.** With `AND t.order_id BETWEEN <batch min> AND <batch max>` in the `ON` clause, the matching scan dropped by 71%, and the whole MERGE read about 80% less than on the unclustered table. On the unclustered table the same range barely helped (16 files instead of 18), because every file spans the whole key range.

`<batch min>` and `<batch max>` aren't something Spark works out for you. Compute them from the batch first, then put them into the MERGE as literal values, so Spark can compare them against each file's min/max statistics before it reads anything:

```python
from pyspark.sql import functions as F

lo, hi = spark.table("staging.order_batch").agg(F.min("order_id"), F.max("order_id")).first()

spark.sql(f"""
    MERGE INTO silver.orders t
    USING staging.order_batch s
    ON t.order_id = s.order_id
       AND t.order_id BETWEEN {lo} AND {hi}
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *
""")
```

Two cautions:

1. The range must cover every key in the batch that could match an existing row, or that row will be treated as new and inserted twice.
2. And this is one table and one batch shape: if your batches update keys scattered across the whole table, the changed rows sit in most files anyway and the benefit shrinks.

**Small tables:** clustering a table that fits in one file does nothing, but it doesn't cost anything either, since Fast Optimize turns the OPTIMIZE into a no-op. You do still have to choose keys, so leaving a static lookup table unclustered is perfectly reasonable. I've considered building something custom that decides to cluster a table, or not, based on size or file count, but tables grow and it adds dev effort and complexity for little benefit.

**Partition only for concurrent DML on disjoint partitions.** My original guide recommended liquid clustering everywhere except Bronze, and didn't say when partitioning is still the right call. The answer is narrower than "multi-source ingestion": blind appends don't conflict under Delta's concurrency control, with or without partitions. The real exception is concurrent writes that update or delete existing rows (UPDATE, DELETE, or a MERGE that does either) on disjoint data. Cole, at the [AMA][ama]: *"Partitioning is really the only way to guarantee that disjoint DML predicates won't conflict."*

Before reaching for partitions, see whether you can avoid the concurrency instead: run those writes in sequence, or combine the sources into a single MERGE. Partitioning is for when you genuinely need parallel DML on the same table. If you're in that case:

- Use low-to-moderate cardinality columns, with [at least 1 GB per partition](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-partitioning#when-to-use-partitioning).
- Put the partition column [**in the MERGE `ON` condition itself**](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-partitioning#partitioning-and-concurrent-writes). Having it in the source data isn't enough to prevent conflicts.
- Use Z-Order within partitions if you also need skipping. Liquid clustering can't be combined with partitioning.

**Use `OPTIMIZE FULL` after changing cluster keys**, or once on Runtime 1.3-era tables, as above:

```sql
ALTER TABLE gold.sales CLUSTER BY (store_id, sale_date);
OPTIMIZE gold.sales FULL;
```

---

## Deletion Vectors

### How it works

Without deletion vectors, any command that deletes or updates a row (DELETE, UPDATE, or a MERGE that does either) rewrites the entire Parquet file containing it. With deletion vectors, the affected rows are marked as deleted in a small sidecar file and the Parquet file is left alone. That makes writes much cheaper, at the cost of readers having to apply the deletion vectors until compaction rewrites the file.

Direct Lake is where the cost shows: on cold start it has to load every deletion vector for the table, so accumulated deletion vectors directly slow the first queries your users run. Now that deletion vectors are on by default, every Direct Lake model over new Runtime 2.0 tables is exposed to this, whether or not anyone chose to enable them.

### What changed in Runtime 2.0

- **Deletion vectors are [on by default](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-deletion-vectors) for new tables.** My original guide suggested enabling them layer by layer, but on Runtime 2.0 there's nothing to enable. New tables are created at protocol reader 3/writer 7 with `deletionVectors` as a table feature.
- **Only new tables.** A table I created on Runtime 1.3 still had no deletion vectors after moving to Runtime 2.0 and running OPTIMIZE there. If you want deletion vectors on older tables, enable them yourself (and read the compatibility notes first).

### What to do

**Keep them on.** Then manage the build-up:

- **OPTIMIZE [purges a file's deletion vectors](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#prevent-and-compact-small-files) once more than 5% of its rows are deleted.**
- **Auto Compaction only purges them when its small-file trigger also fires.** Microsoft [says so explicitly](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#prevent-and-compact-small-files): *"If a workload performs updates or deletes without generating small files, periodically run OPTIMIZE."* I confirmed it. Three 10% UPDATEs on an Auto Compaction table each added deletion vectors, and no Auto Compaction ran because the table never reached 50 small files. A manual OPTIMIZE then cleared them.

So update-heavy MERGE targets, whether at Silver, Gold or merge-style Bronze, can build up deletion vectors indefinitely under Auto Compaction alone. **Schedule an OPTIMIZE on them**, and for Direct Lake tables, run it straight after the loads that create the deletion vectors (see the [timing note in Compaction](#compaction)).

To spot the build-up, look at `DESCRIBE HISTORY`:

- UPDATE and DELETE report `numDeletionVectorsAdded`
- MERGE reports `numTargetDeletionVectorsAdded`
- OPTIMIZE reports `numDeletionVectorsRemoved`

(Don't read UPDATE cost from `numRemovedBytes`, as on the deletion-vector path it reports 0.)

[`REORG TABLE ... APPLY (PURGE)`](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-deletion-vectors#use-reorg-purge-to-remove-accumulated-deletion-vectors) forces deletion vectors out regardless of the 5% threshold. Use it for compliance-driven hard deletes, not routine maintenance.

**Compatibility:** deletion vectors upgrade the table protocol, and not every reader supports them. In particular, [Python notebooks (`delta-rs`, Polars) can't read them](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability#current-limitations). If you rely on those readers, you now have to **turn deletion vectors off explicitly** on the new tables they read, because Runtime 2.0 turns them on:

```sql
CREATE TABLE silver.my_table (...) TBLPROPERTIES ('delta.enableDeletionVectors' = 'false');
```

(Or set `spark.databricks.delta.properties.defaults.enableDeletionVectors = false` in the writing session.) And see the [Data Factory section](#data-factory): Copy's merge writer can add deletion vectors to a table even after you've turned them off.

---

## Vacuuming and Retention

### How it works

OPTIMIZE, UPDATE, DELETE and MERGE all leave old files behind. That's what makes time travel possible. [VACUUM](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-vacuum) deletes unreferenced files older than the retention window, which defaults to 7 days. OPTIMIZE improves performance, VACUUM is the step that actually reduces your storage bill.

```sql
VACUUM gold.my_table DRY RUN;   -- see what would be removed
VACUUM gold.my_table;
```

### What changed in Runtime 2.0

[**VACUUM LITE**](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-vacuum) finds unreferenced files by reading the Delta log rather than listing the whole directory, which is much cheaper on large tables. It's easy to assume it quietly falls back to a full VACUUM when it can't run, but that would be too convenient! If the log doesn't have enough history, it raises `DELTA_CANNOT_VACUUM_LITE`, and you need to handle that:

```python
def vacuum(table):
    try:
        spark.sql(f"VACUUM {table} LITE")
    except Exception as e:
        if "DELTA_CANNOT_VACUUM_LITE" in str(e):
            spark.sql(f"VACUUM {table}")      # fall back to a full directory listing
        else:
            raise
```

For very large tables there's also **`VACUUM ... USING INVENTORY`**, where you supply a precomputed file listing instead of having VACUUM list the directory.

### What to do

**Set retention per table, with the right property.** It's important to note that `delta.logRetentionDuration` is the transaction log's retention ([default 30 days](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-time-travel#understand-retention-and-availability)), and it doesn't change what VACUUM deletes. VACUUM retention is `delta.deletedFileRetentionDuration`. I tested both:

- Setting the file property to 14 days changed VACUUM's retention to 14 days.
- Setting the log property to 60 days left VACUUM at 7.

Time travel can only go as far back as the minimum of the two (once VACUUM has run), so to keep 30 days of time travel, set both to 30 days:

```sql
ALTER TABLE silver.my_table SET TBLPROPERTIES (
  'delta.deletedFileRetentionDuration' = 'interval 30 days',
  'delta.logRetentionDuration'         = 'interval 30 days'
);
```

| Layer | Retention | Reason |
|---|---|---|
| Bronze | 7 days (default) | Raw data, time travel rarely needed |
| Silver | 14–30 days | Supports debugging and rollback of transformations |
| Gold | 7–14 days | Longer if Change Data Feed consumers are active |

If anything reads a table's Change Data Feed, keep retention comfortably longer than your slowest consumer's polling interval, or it will miss changes.

Run VACUUM on its own schedule, after OPTIMIZE: weekly is usually enough. Only shorten retention below 7 days deliberately. The safety check (`spark.databricks.delta.retentionDurationCheck.enabled`) exists for a reason.

---

## V-Order and Resource Profiles

### How it works

V-Order applies VertiPaq-style sorting and encoding to Parquet files at write time. Direct Lake benefits most. The cost is write time: Microsoft's [V-Order page](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order#what-is-v-order) puts it at *"often around 15% on average."* It's off by default.

### What changed in Runtime 2.0

Nothing in the runtime, but the guidance moved:

- Microsoft [now says](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#apply-the-guidance-to-medallion-layers) to *"enable V-Order only when Direct Lake is a primary consumer"*, and explicitly not for SQL analytics endpoint performance alone. My original guide recommended it for SQL endpoint consumers too, so drop that.
- The [`readHeavyForPBI` resource profile](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-resource-profile-configurations) is now [presented](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#cross-workload-guidance) as an equal alternative to enabling V-Order yourself.
- My original guide quoted 40–60% Direct Lake cold-cache gains and ~10% for the SQL endpoint. Those came from the Cross-Workload page and went in [the same August revision](https://github.com/MicrosoftDocs/fabric-docs/commit/2e2cc1c5e8c6f536d95e0066da132ab46380cdbf), and no current Learn page gives a Direct Lake figure, so I've removed them.

The documentation disagrees on resource profiles: the [V-Order page](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order#control-v-order-writes) says `readHeavyForSpark` enables V-Order, while the [Resource Profiles page](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-resource-profile-configurations) says only `readHeavyForPBI` does. I checked in a session:

| Profile | V-Order | Optimize Write | Optimize Write bin size |
|---|---|---|---|
| `writeHeavy` (default) | Off | Unset (partitioned tables only) | 128 MB |
| `readHeavyForSpark` | **Off** | On | 128 MB |
| `readHeavyForPBI` | **On** | On | 1 GB |

`readHeavyForPBI` is the only profile that turns on V-Order.

### What to do

| Layer | V-Order |
|---|---|
| Bronze | Off |
| Silver | Only for tables Direct Lake reads directly (table property) |
| Gold | On for Direct Lake tables, or use `readHeavyForPBI` |

```sql
ALTER TABLE silver.my_table SET TBLPROPERTIES ('delta.parquet.vorder.enabled' = 'true');
```

---

> ### Direct Lake considerations
> - **Deletion vectors slow cold start, and they're now on by default.** Direct Lake loads them all. Purge them with OPTIMIZE.
> - **Time OPTIMIZE around framing.** With automatic updates on (the default), the model reframes whenever a table changes, so run OPTIMIZE straight after the load. If you've turned automatic updates off, finish OPTIMIZE before your scheduled refresh.
> - **Row groups:** Microsoft [says](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#power-bi-direct-lake) Direct Lake *"generally performs best with row groups between 1 million and 16 million rows."* There's a native-writer setting, `spark.sql.parquet.native.writer.maxRowGroupRowCount`, but Microsoft advises touching it only after analysis shows row-group sizing is the problem.
> - **My reasoning, not a documented claim:** every rewritten file invalidates its transcoded VertiPaq segments. On Runtime 1.3, every OPTIMIZE on a liquid clustered table under 100 GB rewrote the whole table and therefore invalidated all of it. Incremental clustering should mean far less re-transcoding, not just less Spark work.
> - Use [**Delta Analyzer**](https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-understand-storage#analyzing-delta-table-updates) to look at row groups and segments when a model is slow.

---

## Assessing Your Tables

If you've never maintained your tables, you *can* just run OPTIMIZE on everything: on Runtime 2.0, Fast Optimize makes it nearly free on tables that are already healthy. The reason to look first is the unhealthy ones. A neglected table's first OPTIMIZE rewrites most of it, and a long list of those run back-to-back is a large CU spike on a shared capacity. An assessment tells you where those big rewrites are, so you can prioritise and stagger them, and it surfaces partitioned tables and old protocols worth revisiting. This snippet reads metadata only, so it runs in seconds regardless of table size. It compares each table against **its own ATFS target** rather than a fixed per-layer constant, and it shows each table's clustering and protocol.

```python
from pyspark.sql import functions as F, types as T

def to_mb(value, default=128.0):
    """Parse a Delta size string such as '128m' or '1g' into MB."""
    if not value:
        return default
    v = str(value).strip().lower()
    units = {"k": 1 / 1024, "m": 1.0, "g": 1024.0}
    return float(v[:-1]) * units[v[-1]] if v[-1] in units else float(v) / 1_048_576

rows = []
# Schema-enabled Lakehouses: SHOW TABLES only lists the current schema, so loop over every schema
tables = [f"`{s[0]}`.`{t.tableName}`"
          for s in spark.sql("SHOW SCHEMAS").collect()
          for t in spark.sql(f"SHOW TABLES IN `{s[0]}`").collect()
          if not t.isTemporary]
for name in tables:
    try:
        d = spark.sql(f"DESCRIBE DETAIL {name}").first().asDict()
        props = d.get("properties") or {}
        n, size = d.get("numFiles") or 0, d.get("sizeInBytes") or 0
        avg_mb = round(size / n / 1_048_576, 1) if n else 0.0
        target_mb = to_mb(props.get("delta.targetFileSize.adaptive") or props.get("delta.targetFileSize"))
        if n <= 1:
            status = "Skip: single file"
        elif avg_mb >= target_mb / 2:
            status = "Healthy"
        elif avg_mb >= target_mb / 4:
            status = "Review"
        else:
            status = "Needs OPTIMIZE"
        rows.append((name, n, round(size / 1_073_741_824, 3), avg_mb, target_mb,
                     ", ".join(d.get("clusteringColumns") or []) or None,
                     bool(d.get("partitionColumns")),
                     f"{d['minReaderVersion']}/{d['minWriterVersion']}",
                     "deletionVectors" in (d.get("tableFeatures") or []),
                     status))
    except Exception as e:
        rows.append((name, None, None, None, None, None, None, None, None, f"Error: {e}"))

schema = T.StructType([
    T.StructField("table", T.StringType()),
    T.StructField("num_files", T.LongType()),
    T.StructField("size_gb", T.DoubleType()),
    T.StructField("avg_file_mb", T.DoubleType()),
    T.StructField("target_mb", T.DoubleType()),
    T.StructField("cluster_by", T.StringType()),
    T.StructField("partitioned", T.BooleanType()),
    T.StructField("protocol", T.StringType()),
    T.StructField("deletion_vectors", T.BooleanType()),
    T.StructField("status", T.StringType()),
])
display(spark.createDataFrame(rows, schema).orderBy(F.col("avg_file_mb").asc_nulls_last()))
```

| Column | What it tells you |
|---|---|
| `avg_file_mb` vs `target_mb` | Files averaging under half the table's own target suggest fragmentation. An average can hide skew, so treat "Review" as a prompt to look closer. |
| `cluster_by`/`partitioned` | Partitioned tables without concurrent-DML needs are candidates for liquid clustering. |
| `protocol`/`deletion_vectors` | `1/2` or no deletion vectors usually means a table created before Runtime 2.0 or by a non-Spark writer. |
| `status` | Your triage order. |

For deletion-vector build-up, check history since the last OPTIMIZE:

```python
from pyspark.sql import functions as F

def dvs_since_last_optimize(table):
    added = 0
    for r in spark.sql(f"DESCRIBE HISTORY {table}").orderBy(F.desc("version")).collect():
        if r.operation == "OPTIMIZE":
            break
        m = r.operationMetrics or {}
        added += int(m.get("numDeletionVectorsAdded", 0)) + int(m.get("numTargetDeletionVectorsAdded", 0))
    return added
```

Treat this as a trend indicator rather than an exact count of outstanding deletion vectors. It stops at the most recent OPTIMIZE commit, including Auto Compaction runs, and OPTIMIZE only purges files with more than 5% of rows deleted, so some deletion vectors can survive it. A number that keeps climbing between runs is the signal to schedule an OPTIMIZE.

For liquid clustered tables, `clusteringQuality()` (Scala) reports, per cluster column, the number of files, average and maximum depth (how many files overlap a given value, 1 is ideal), overlap ratio (0 is ideal) and skipping effectiveness (1 is ideal):

```scala
%%spark
import io.delta.tables.DeltaTable
display(DeltaTable.forName(spark, "silver.orders").clusteringQuality())
```

It's a good reminder that "clustered" doesn't mean "skippable". On a small test table with one clustered file and ten unclustered appends, every file spanned the full key range: `avg_depth 11`, `overlap_ratio 1`, `skipping_effectiveness 0`. No query on that key could skip a single file until OPTIMIZE clustered the new data.

**Clearing a backlog:** run a one-off OPTIMIZE on the "Needs OPTIMIZE" tables (Silver and Gold first), VACUUM them, then enable Auto Compaction so you don't end up back here. That's also the order [Microsoft's remediation guidance](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#resolve-layout-and-maintenance-issues) gives.

---

## Implementation Guide

### Your utility notebook on Runtime 2.0

It gets much shorter:

```python
# Auto Compaction: still off by default. Prefer the table property
# (delta.autoOptimize.autoCompact) for tables written by more than one notebook or tool.
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")

# Evaluate Auto Compaction at checkpoints (every 10 commits) instead of after every commit
spark.conf.set("spark.microsoft.delta.autoCompact.onCheckpointOnly.enabled", "true")

# V-Order: explicit baseline; override per table or in Gold notebooks
spark.conf.set("spark.sql.parquet.vorder.default", "false")
```

What's gone, and why:

- **ATFS, File-Level Target and Fast Optimize**: on by default in Runtime 2.0. Keep them only if the same notebook also runs on 1.3.
- **`optimizeWrite.enabled = true`**: forcing it overrides `writeHeavy`'s partitioned-only behaviour for every table. Leave it unset and enable it per table where it helps.

### Pipeline order

1. Load.
2. OPTIMIZE where you've decided to schedule it (update-heavy MERGE targets, backlog tables).
3. Direct Lake picks up the result (automatically, or at your scheduled refresh if automatic updates are off).
4. VACUUM on its own, slower cadence.

### If your pipelines write Delta tables with Data Factory

This one surprised me, and it's the most practical Runtime 2.0 issue I found.

> **Copy activity Upsert and Copy job Merge fail on tables created by Spark on Runtime 2.0.**
>
> Every table Spark creates on Runtime 2.0 is written at protocol writer 7, and Spark explicitly lists the legacy `appendOnly` feature in it. The Delta writer used by Copy's Upsert and Merge (`Microsoft.DI.Delta`) doesn't support that feature and rejects the table:
>
> ```
> ErrorCode=FailedToUpsertDataIntoDeltaTable ... UnsupportedTableFeatureException,
> Message=Table requires writer feature(s) 'appendOnly' which are not supported by this SDK version.
> ```
>
> - It's not the deletion vectors themselves. Copy's writer handles those.
> - It affects **every Spark-created Runtime 2.0 table**, and any Runtime 1.3 table where you enabled deletion vectors (or another table feature) from Spark.
> - It also affects **every liquid clustered table**, regardless of who created it (`'domainMetadata', 'clustering'`).
> - **Copy job's Merge fails identically.** That's the tool [Microsoft recommends](https://learn.microsoft.com/en-us/fabric/data-factory/what-is-copy-job) for incremental loads.
> - `appendOnly` can't be dropped (`DELTA_FEATURE_DROP_NONREMOVABLE_FEATURE`), so once a table lists it, Copy can never upsert into it.
> - Failed runs write nothing. Append still works.

What I'd do instead, and what my team already does: **use Copy to land files (for example Parquet in the Lakehouse `Files` area), then MERGE into the Delta table from a Spark notebook.** Copy never writes Delta, so none of this applies, and Spark is the only engine writing each table.

If you'd rather not hand-write MERGEs, the same principle works with declarative tools that run on Spark: dbt incremental models through the [Fabric Spark adapter](https://github.com/microsoft/dbt-fabricspark), or [Materialized Lake Views](https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/overview-materialized-lake-view), which can even ingest the landed CSV or Parquet files directly. What matters is that Spark, not Copy, writes the Delta tables. One catch with Materialized Lake Views: they only refresh incrementally when their source Delta tables have Change Data Feed enabled, otherwise each refresh is either skipped or a full rebuild.

A few other things I found while testing Copy, all on 2 October 2026:

- **Copy's merge writer writes deletion vectors even when the table says not to.** On a table with `delta.enableDeletionVectors = false`, a Copy activity Upsert upgraded the protocol and wrote deletion vectors. On a Copy job–created table where the property isn't set, an incremental Merge did the same. The [Delta protocol](https://github.com/delta-io/delta/blob/master/PROTOCOL.md#deletion-vectors) is explicit: *"Writers must only write new Deletion Vectors (DVs) when this property is set to `true`."* Spark reads the data correctly either way. But if you turned deletion vectors off on a table so that `delta-rs`, Polars or an external reader can use it, a Copy Upsert or Merge can switch it to deletion vectors anyway, and those readers will start failing.
- **Copy changes table properties.** On the table it upgraded, and on tables Copy job creates, it set `delta.logRetentionDuration` to 168 hours: 7 days of history instead of the [30-day default](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-time-travel#understand-retention-and-availability). It only seemed to fill in properties that weren't already set (it left an explicitly enabled Change Data Feed alone), so a retention you've set explicitly should survive, though I didn't test that property specifically. Tables relying on the *default* 30 days lose 23 days of history and time travel without warning. If that matters to you, set retention explicitly on every table, and check tables Copy has written to.
- **Copy-created tables use a different protocol.** Copy activity creates tables at `1/2` without deletion vectors, so a single Lakehouse can end up with a mix of protocols depending on which tool created each table.
- **The documentation contradicts itself here too.** The [Delta Lake interoperability matrix](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability#delta-lake-features-and-fabric-experiences) says Pipelines don't support deletion vectors and write `1/2`, while the [Copy activity page](https://learn.microsoft.com/en-us/fabric/data-factory/connector-lakehouse-copy-activity#delta-lake-table-support) says deletion vectors and liquid clustering are supported as a destination. Neither is quite right.

### Other compatibility notes

- **Python notebooks** (`delta-rs`, Polars) can't write or OPTIMIZE liquid clustered tables, can't read deletion vectors, and won't for some time. Cole put `delta-rs` support at "multiple years" away ([AMA][ama], [delta-rs#2043](https://github.com/delta-io/delta-rs/issues/2043)). Use PySpark notebooks for anything touching these tables.
- **Dataflow Gen2** can [read liquid clustered tables but not write them](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability#delta-lake-features-and-fabric-experiences). Its incremental-refresh destinations [don't support OPTIMIZE or REORG](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization#use-the-spark-runtime-defaults).
- **OPTIMIZE is Spark-only.** The SQL analytics endpoint and Warehouse can't run it.
- [**Concurrency**](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-concurrency-control): OPTIMIZE uses snapshot isolation, so it doesn't conflict with concurrent blind INSERTs. Two OPTIMIZEs, or OPTIMIZE plus DML on the same files, do conflict. Auto Compaction avoids this by running in the writer's own session, which is another reason to prefer it over a separately scheduled job.

---

## Upgrading from Runtime 1.3: Checklist

- [ ] **Switch the workspace default or Environment item to Runtime 2.0**, and **publish** the Environment. An unpublished runtime change silently leaves your sessions on the old runtime, as I discovered. Confirm with `print(spark.version)`: Runtime 2.0 is Spark 4.1.x, Runtime 1.3 is 3.5.x.
- [ ] [**Python 3.13 breaks Environment libraries.**](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0) Remove the libraries, publish, re-add them, publish again.
- [ ] **Remove the 1.3 workarounds:** ATFS, File-Level Target and Fast Optimize settings (and make sure nothing sets them to `false`), and any `autoCompact.enabled = false` overrides you added for liquid clustered tables.
- [ ] **If anything reads your tables with `delta-rs` or Polars**, turn deletion vectors off on the new tables it reads.
- [ ] **Liquid clustered tables need no migration.** Optionally run `OPTIMIZE FULL` once on large or heavily queried ones to replace Runtime 1.3's layout.
- [ ] **Decide about deletion vectors on existing tables.** Runtime 2.0 only enables them on new tables.
- [ ] **Check pipelines that Upsert or Merge with Copy activity or Copy job** into tables Spark creates or alters.
- [ ] **Re-evaluate partitioned tables** for liquid clustering (keep partitioning only for concurrent DML on disjoint data).
- [ ] **Remove `delta.targetFileSize` pins** you can't justify.
- [ ] **Run the defaults snippet** [from the top of this guide](#runtime-2-0-at-a-glance) in your own environment.

---

## What's Still Your Decision

Runtime 2.0 has made most of the old configuration decisions for you. These are the ones it hasn't:

- **Liquid clustering, Z-Order or partitioning.** Liquid clustering by default, partitioning only for concurrent DML on disjoint data.
- **Auto Compaction or scheduled OPTIMIZE.** Auto Compaction by default, plus scheduled OPTIMIZE for update-heavy MERGE targets, timed around Direct Lake framing.
- **V-Order.** Only where Direct Lake reads the table.
- **Native Execution Engine.** Turn it on.

---

## The Cheat Sheet

| Layer | Auto Compaction | Optimize Write | V-Order | Clustering | Deletion vectors | Scheduled OPTIMIZE | VACUUM retention |
|---|---|---|---|---|---|---|---|
| Bronze | On (table property) | Default, on for streaming | Off | Liquid clustering on the merge key for MERGE targets | Keep (default) | Update-heavy MERGE targets | 7 days |
| Silver | On (table property) | Default, on for streaming | Only where Direct Lake reads it | Liquid clustering by default | Keep (default) | Update-heavy MERGE targets | 14–30 days |
| Gold | On (table property) | Default | On for Direct Lake, or `readHeavyForPBI` | Liquid clustering by default | Keep, purge with OPTIMIZE | Straight after loads (before scheduled refresh if automatic updates are off) | 7–14 days |

---

## Where the Docs Disagree

Microsoft's documentation still contradicts itself in places. Here are the conflicts I found, and what the runtime actually does:

| Topic | One page says | Another says | What I measured |
|---|---|---|---|
| File-Level Compaction Target default | "Not enabled by default" ([Table Compaction](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#file-level-compaction-targets)) | "Enabled by default starting in Runtime 2.0" ([Tune File Size](https://learn.microsoft.com/en-us/fabric/data-engineering/tune-file-size#understand-performance-impact)) | On by default |
| Fast Optimize on liquid clustered tables | "Not applicable to liquid clustering" ([Table Compaction](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction#fast-optimize)) | "Compatible starting in Runtime 2.0" ([Liquid Clustering](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#interaction-with-other-features)) | It applies, and skips small bins |
| Fast Optimize default | Not stated on Learn | On by default (Cole, [AMA][ama]) | On by default |
| Delta version in Runtime 2.0 | Delta 4.1 ([Liquid Clustering](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering#incremental-liquid-clustering), [VACUUM](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-vacuum) pages) | Delta 4.2 ([Runtime 2.0 page](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0)) | Delta 4.2.0 |
| `readHeavyForSpark` and V-Order | Enables V-Order ([V-Order page](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order#control-v-order-writes)) | Doesn't ([Resource Profiles page](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-resource-profile-configurations)) | Doesn't |
| Pipelines and deletion vectors | Not supported ([interoperability matrix](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability#delta-lake-features-and-fabric-experiences)) | Supported as source and destination ([Copy activity page](https://learn.microsoft.com/en-us/fabric/data-factory/connector-lakehouse-copy-activity#delta-lake-table-support)) | Read and Append work, Upsert/Merge fail on Spark-created tables |
| Runtime 2.0 as default for new workspaces | "Planned" ([Runtime 2.0 page](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0)) | No other page | Already the default |


---

## What to Watch

- **Lakehouse Maintenance activity** losing its schema-enabled and Private Link limitations.
- **An automatic maintenance option**, along the lines of Databricks' [predictive optimisation](https://docs.databricks.com/aws/en/optimizations/predictive-optimization).
- **Copy activity and Copy job writer support** for `appendOnly` and liquid clustering features.
- **Auto Compaction becoming less synchronous.** Cole mentioned plans for this at the [AMA][ama].
- **No plans for `CLUSTER BY AUTO`**, per Cole at the [AMA][ama].
- **`delta-rs` liquid clustering support** for Python notebooks ([delta-rs#2043](https://github.com/delta-io/delta-rs/issues/2043)).

---

## Where the Documentation Lives

**Fabric maintenance and configuration:**
- [Cross-Workload Table Maintenance and Optimization](https://learn.microsoft.com/en-us/fabric/fundamentals/table-maintenance-optimization): Microsoft's overall strategy. It was revised in August and September 2026, so anything quoting its earlier 400 MB targets or "run OPTIMIZE aggressively", including my original guide, cites superseded guidance.
- [Table Compaction](https://learn.microsoft.com/en-us/fabric/data-engineering/table-compaction): Auto Compaction, Fast Optimize, File-Level Target
- [Tune File Size](https://learn.microsoft.com/en-us/fabric/data-engineering/tune-file-size): ATFS, Optimize Write
- [Liquid Clustering](https://learn.microsoft.com/en-us/fabric/data-engineering/liquid-clustering): incremental clustering, Runtime 1.3 vs 2.0
- [Partitioning for Delta tables](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-partitioning)
- [Concurrency control for Delta tables](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-concurrency-control)
- [Deletion vectors for Delta tables](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-deletion-vectors)
- [VACUUM Delta tables](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-vacuum)
- [Query Delta tables with time travel](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-time-travel): retention defaults
- [Delta Optimization and V-Order](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order)
- [Resource Profile Configurations](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-resource-profile-configurations)
- [Runtime 2.0 in Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime-2-0)
- [Delta Lake table format interoperability](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability)
- [Lakehouse Maintenance Activity](https://learn.microsoft.com/en-us/fabric/data-factory/lakehouse-maintenance-activity)
- [File skipping for Delta tables](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-file-skipping): statistics on the first 32 columns
- [How Direct Lake works](https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-how-it-works): framing and automatic updates
- [Materialized Lake Views](https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/overview-materialized-lake-view)
- [Configure Lakehouse in a copy activity](https://learn.microsoft.com/en-us/fabric/data-factory/connector-lakehouse-copy-activity) and [What is Copy job](https://learn.microsoft.com/en-us/fabric/data-factory/what-is-copy-job)

**From the Fabric team:**
- [Incremental Liquid Clustering in Fabric Runtime 2.0](https://milescole.dev/data-engineering/2026/07/24/Incremental-Liquid-Clustering-Fabric-Runtime-2.html), Miles Cole's deep dive, and his [interactive liquid clustering simulator](https://milescole.dev/playground/incremental-liquid-clustering/)
- [Incremental Liquid Clustering in Microsoft Fabric: Faster, smarter, and truly incremental](https://community.fabric.microsoft.com/t5/Fabric-Updates-Blog/Incremental-Liquid-Clustering-in-Microsoft-Fabric-Faster-smarter/ba-p/5189122), Miles Cole (27 May 2026)
- [Introducing Optimized Compaction in Fabric Spark](https://community.fabric.microsoft.com/t5/Fabric-Updates-Blog/Introducing-Optimized-Compaction-in-Fabric-Spark/ba-p/5172547), Miles Cole (October 2025)
- [Microsoft Fabric Table Maintenance Optimization: A Cross-Workload Survival Guide](https://christopherfinlan.com/2026/02/15/microsoft-fabric-table-maintenance-optimization-a-cross-workload-survival-guide/), Christopher Finlan (February 2026, predates Runtime 2.0 GA)
- [Fabric Spark team AMA][ama] on r/MicrosoftFabric (25 August 2026), and Miles Cole's [incremental liquid clustering thread][cole-reddit] (May 2026)
- [Docs change: "Simplify data layout guidance based on benchmarking"](https://github.com/MicrosoftDocs/fabric-docs/commit/2e2cc1c5e8c6f536d95e0066da132ab46380cdbf), the 26 August revision that removed the fixed file-size targets

**Open source:**
- [Delta Lake protocol specification](https://github.com/delta-io/delta/blob/master/PROTOCOL.md)
- [Delta Lake utility commands](https://docs.delta.io/latest/delta-utility.html)
- [Delta Lake liquid clustering](https://docs.delta.io/delta-clustering/): cluster-key limits and statistics requirement

---

## Final Thoughts

Runtime 2.0 is the release where Fabric's maintenance story grew up. Most of the configuration I told you to set is now the default, liquid clustering finally works the way it was always meant to, and the documentation, while still contradicting itself in places, is far better than it was a year ago.

Three things I'd leave you with:

**First, rewrite your utility notebook.** Most of it is now redundant. Enable Auto Compaction (as a table property where you can), keep an explicit V-Order baseline, and remove the rest.

**Second, use liquid clustering by default, and keep a scheduled OPTIMIZE for the tables that need it.** On my test table, Runtime 2.0 rewrote 1–3% of the table where Runtime 1.3 rewrote all of it. Migration is automatic. But Auto Compaction still doesn't purge deletion vectors from update-heavy MERGE targets on its own, so those still need a scheduled OPTIMIZE.

**Third, test what the documentation tells you.** In a day of testing I found seven places where Microsoft's own pages disagreed with each other or with the runtime, and a Data Factory incompatibility that will break pipelines upserting into Runtime 2.0 tables. None of it is hard to check.

And as before: treat maintenance as cost control, not just performance. Fabric SKUs double at every tier, and the difference between a well-maintained platform and a neglected one may be the difference between your current SKU and the next one up.

---

*Brad Coles is an Associate Director and Data Engineering Capability Lead ANZ at Synechron Australia, specialising in Microsoft Fabric and modern data platform engineering. [linkedin.com/in/brad-coles](https://www.linkedin.com/in/brad-coles/)*

<!-- Link references -->
[ama]: <https://www.reddit.com/r/MicrosoftFabric/comments/1vsw40t/experts_engines_hi_were_the_microsoft_fabric/> "r/MicrosoftFabric: Experts + Engines, Fabric Spark team AMA (25 Aug 2026)"
[cole-reddit]: <https://www.reddit.com/r/MicrosoftFabric/comments/1tpio6g/announcing_incremental_liquid_clustering/> "r/MicrosoftFabric: Announcing Incremental Liquid Clustering (u/mwc360, May 2026)"
[f0]: <https://community.fabric.microsoft.com/blog/fbc_fabricupdatesblogs/fabric-september-2026-feature-summary/5325825> "Fabric September 2026 Feature Summary"
