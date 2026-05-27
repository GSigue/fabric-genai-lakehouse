# Monday Checklist — Project 1 Kickoff

**Target: 3–4 hours · End-state: first commit pushed, Fabric workspace live, data sourced**

## Block 1 — Repo setup (45 min)

- [ ] Create GitHub repo `fabric-genai-lakehouse`, public, MIT license
- [ ] Clone locally
- [ ] Drop these files in:
  - [ ] `README.md` (from the kit)
  - [ ] `.gitignore` (from the kit)
  - [ ] `LICENSE` (MIT — GitHub generates this for you on creation)
- [ ] Create folder structure:
  ```
  /notebooks
  /sql
  /dax
  /semantic-model
  /docs/images
  /powerbi
  ```
- [ ] Add `dax/measures.md` (from the kit)
- [ ] Add `docs/architecture.md` (from the kit)
- [ ] **First commit:** `"Initial repo structure with README and architecture diagram"`
- [ ] Push.

## Block 2 — Fabric workspace (45 min)

- [ ] Sign in to app.fabric.microsoft.com
- [ ] Start Fabric trial if not already on capacity
- [ ] Create workspace `lakehouse-portfolio-v1` and assign trial capacity to it
- [ ] In the workspace, create:
  - [ ] Lakehouse named `bronze`
  - [ ] Lakehouse named `silver`
  - [ ] Warehouse named `gold`
- [ ] Confirm all three appear in the workspace home

## Block 3 — Data sourcing (60–90 min)

- [ ] Decide on slice: **Yellow Taxi, Jan–Jun 2024** (6 monthly parquet files, ~3GB total)
- [ ] Source URL pattern (verify on the TLC site Monday morning — they change paths occasionally):
  `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet`
  through `yellow_tripdata_2024-06.parquet`
- [ ] Also grab the taxi zone lookup: `taxi_zone_lookup.csv` from the NYC TLC trip record data page
- [ ] In Bronze Lakehouse, create a `Files/raw` folder
- [ ] Upload the 6 parquet files + zone lookup CSV
  - Option A: drag-and-drop in the Fabric UI
  - Option B: a quick PySpark notebook cell that downloads with `requests` and writes to OneLake — but UI upload is faster for one-time seeding

## Block 4 — Notebook scaffolding (30 min)

- [ ] In the workspace, create three empty notebooks:
  - [ ] `01_bronze_ingest`
  - [ ] `02_silver_transform`
  - [ ] `03_gold_dimensional`
- [ ] Attach `01_bronze_ingest` to the bronze Lakehouse as default
- [ ] Add a top markdown cell to each titled with the notebook name + one-line purpose
- [ ] Export each as `.ipynb` to your local repo `/notebooks` folder
- [ ] **Second commit:** `"Add empty notebook scaffolds for bronze/silver/gold"`
- [ ] Push.

## End-of-Monday state

By bedtime Monday you should have:
- ✅ 2 commits on GitHub, README visible, MIT license, clean folder structure
- ✅ Fabric workspace named, three items created (bronze/silver/gold)
- ✅ Raw data uploaded to OneLake under `bronze/Files/raw`
- ✅ Three empty notebook shells ready for Tuesday

**No PySpark code written yet.** That's Tuesday. Resist the urge to start ingesting tonight — Monday is foundation. Notebook code on Monday eats Tuesday's budget and you'll fall behind by Wednesday.

---

## Tuesday preview

I'll have the full Bronze ingestion notebook drafted Tuesday morning before you sit down: read parquet from `Files/raw`, write Delta partitioned by year/month, add audit columns. Your job will be: run it cell by cell, push back where you'd do it differently, commit when it passes.

If anything in Block 1–4 takes longer than estimated, ping me. We have buffer Tuesday, not Monday.
