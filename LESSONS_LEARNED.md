# Lessons Learned

Real issues hit during the build, what was happening underneath, and how they were solved. Written as they happen so the reasoning is honest, not retrofitted.

---

## #1 — Files written through `/lakehouse/default/Files` don't always reach OneLake

**Notebook:** `01_bronze_ingest`

### The symptom

The Bronze notebook downloaded 7 source files (6 monthly TLC parquet files + 1 zone-lookup CSV) using `requests` into `/lakehouse/default/Files/raw`. A POSIX `ls` of the path showed all 7 files present with correct sizes — about 340 MB total.

The very next cell tried to read them with Spark:

```python
raw_df = spark.read.parquet(f"{LANDING_PATH}/yellow_tripdata_*.parquet")
```

This failed with:

```
Py4JJavaError: ... Operation failed: "Bad Request", 400, GET,
http://onelake.dfs.fabric.microsoft.com/...?directory=lakehouse/default/Files/raw...
"Request Failed with WorkspaceId and ArtifactId should be either valid Guids or valid Names"
FriendlyNameSupportDisabled
```

Switching to an explicit ABFSS path (`abfss://<workspace-guid>@onelake.dfs.fabric.microsoft.com/<lakehouse-guid>/Files/raw`) got a different error:

```
AnalysisException: [PATH_NOT_FOUND] Path does not exist: abfss://.../Files/raw/yellow_tripdata_*.parquet
```

A diagnostic listing via `mssparkutils.fs.ls(LANDING_ABFSS)` returned **empty** — the OneLake ABFSS directory existed but had no files in it. Meanwhile, the POSIX `ls` still showed all 7 files present.

To make matters more confusing, when we then tried to copy files from the POSIX path to ABFSS, the local files started disappearing mid-operation. A re-listing showed `2024-06.parquet` missing, and the `cp` loop crashed when it reached `taxi_zone_lookup.csv` with `FileNotFoundError`.

### What's actually going on

Fabric notebooks expose two distinct filesystems that visually look like one:

| Path style | Backed by | Reliable for |
|------------|-----------|--------------|
| `/lakehouse/default/Files/...` | A **local mount on the Spark driver node**, with lazy/best-effort sync to OneLake | Reading already-synced files. Python file I/O *during* a single operation. **Not** durable storage. |
| `abfss://{ws-guid}@onelake.dfs.fabric.microsoft.com/{lh-guid}/...` | Direct OneLake (Azure Data Lake Gen2) | Spark reads/writes. Durable. The source of truth. |

When `requests` (a pure-Python library running on the driver) writes to `/lakehouse/default/Files/...`, the bytes go to the **driver's local disk**, not directly to OneLake. Fabric is supposed to sync that mount to OneLake in the background, but the sync is not transactional, not immediate, and not guaranteed under all conditions. It can fail or lag when:

- Writes come from non-Spark processes (like `requests`).
- The Spark session restarts, autoscales, or evicts the driver before sync completes.
- The tenant has `FriendlyNameSupportDisabled`, which complicates the URL-resolution path the mount uses internally.

The result: files appear on the POSIX mount but are invisible to OneLake-aware tools like `spark.read` or `mssparkutils.fs.ls`. Worse, because the driver disk is ephemeral, those files can be evicted at any time — which is what happened when we tried to copy them. They vanished mid-loop.

### Why the original code looked correct

This pattern — write with `requests` to `/lakehouse/default/Files/...`, then read with Spark — appears in plenty of Fabric tutorials. It usually works because most tenants have friendly-name resolution enabled and the mount sync usually succeeds. "Usually" isn't "always." Tenants with friendly-name disabled, or any case where the sync lags, hit exactly this failure.

### The fix

Bypass the volatile lakehouse mount entirely. Download to the driver's true local scratch (`/tmp`), then explicitly copy to ABFSS using `mssparkutils.fs.cp` with the `file://` scheme:

```python
import urllib.request, os

def download_to_abfss(url, abfss_dest):
    tmp_dir = "/tmp/tlc_staging"
    os.makedirs(tmp_dir, exist_ok=True)
    fname = os.path.basename(abfss_dest)
    tmp_path = f"{tmp_dir}/{fname}"

    urllib.request.urlretrieve(url, tmp_path)
    mssparkutils.fs.cp(f"file://{tmp_path}", abfss_dest, recurse=False)
    os.remove(tmp_path)
```

Three things matter here:

1. **`/tmp`, not `/lakehouse/default/Files`.** `/tmp` is the driver's actual local scratch space. Files written there stay until *we* remove them, not until the lakehouse mount feels like syncing.
2. **Explicit `file://` URI scheme** on the source path in `mssparkutils.fs.cp`. Without it, Hadoop interprets the path as HDFS and can't find the local file. With it, the FileSystem API resolves correctly to the local filesystem.
3. **Destination is ABFSS** (`abfss://<ws-guid>@onelake.../Files/raw`). Once `cp` returns successfully, the file is durably in OneLake. No sync to wait for, no eviction to fear.

### Generalized

For anything that needs to persist in a Fabric Lakehouse, the destination should be the ABFSS path, not the POSIX mount. The POSIX mount (`/lakehouse/default/...`) is convenient for reading already-synced data, but it's a leaky abstraction over OneLake — fine when sync works, dangerous when it doesn't, and impossible to debug from the symptoms alone.

Rule of thumb for the rest of this project:

- **Reads from existing tables** → `spark.read.table("table_name")` or the ABFSS path. Either works because the data is already in OneLake.
- **Writes from PySpark** → `df.write.format("delta").saveAsTable(...)`. Spark handles the OneLake plumbing.
- **Writes from non-Spark Python** (downloads, API responses, file generation) → stage to `/tmp`, then `mssparkutils.fs.cp("file://...", "abfss://...")`. Never write directly to `/lakehouse/default/Files`.

### Finding the ABFSS path for a lakehouse

```python
import notebookutils
lh = notebookutils.lakehouse.getWithProperties("bronze")  # lakehouse name
print(lh["properties"]["abfsPath"])
# → abfss://<workspace-guid>@onelake.dfs.fabric.microsoft.com/<lakehouse-guid>
```

Append `/Files/...` or `/Tables/...` as needed.

The fix has the same shape: write to the explicit ABFSS path under `Tables/` using `.save(f"{LAKEHOUSE_ABFSS}/Tables/yellow_tripdata")` instead of `.saveAsTable("yellow_tripdata")`. The Lakehouse metastore auto-discovers Delta folders under `Tables/` within a minute or two, so the table still shows up in the UI and is queryable by name afterward.