# LinkedIn — delta-doctor launch post

---

After publishing my Delta table maintenance guide, the follow-up was always the same: great, but how do we actually implement this?

Fabric automates Delta maintenance for Warehouses. For Lakehouses, there's nothing built in. Small files accumulate, deletion vectors build up, nothing runs automatically. Left unchecked, it compounds. And without visibility into table health, most teams don't know how bad it's got until query costs spike.

Today I'm releasing delta-doctor: the code that guide was missing. Seven PySpark notebooks for automated Delta table maintenance on Fabric Lakehouses. Free and open source, import directly into your workspace.

What's in it:
- Health scanner: classifies every table in seconds, no data scan
- OPTIMIZE + VACUUM for single tables and full Lakehouses, gated on actual table health
- Rebaseline orchestrator for right-sizing a Lakehouse that's never had maintenance applied
- Session config and table property notebooks to set the right defaults once

Run the health scanner on your Lakehouse today to see what state it's in, without any risk.

Link in comments.

Thanks to @Miles Cole whose Microsoft documentation and blog posts on Delta maintenance in Fabric informed a lot of this.

#MicrosoftFabric #DeltaLake #DataEngineering #OpenSource

---

First comment:
Delta table maintenance guide: https://bradcoles.dev/blog/fabric-delta-table-maintenance.html
Full write-up: https://bradcoles.dev/blog/introducing-delta-doctor.html
Docs and download: https://deltadoctor.dev
GitHub: https://github.com/bradcoles-dev/delta-doctor
