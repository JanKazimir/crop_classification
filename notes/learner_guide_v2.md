# 🛰️ Satellite Crop Type Classification on Azure
### Learner Guide — Wallonia, Belgium 2023

---

> **How to use this guide**
>
> This document walks you through every step of the project — Azure setup, data access, and the pipeline overview.
> When you see a 📓 **Notebook** marker, open `student_workbook.ipynb` and complete the corresponding step there.
> Use the full tutorial document as a reference if you get stuck on the code.

---

## Project Overview

Over 4 days you will build a complete satellite image classification pipeline that maps crop and land cover types across a ~40 × 31 km study area in Wallonia, Belgium.

**What you will produce:**
- A multi-temporal Sentinel-2 feature stack (11 bands × 12 monthly composites + spectral indices)
- A Random Forest classifier trained on official Wallonia field survey data
- A final GeoTIFF map of predicted crop and land cover types

**Study area**

| Parameter | Value |
|---|---|
| Region | Wallonia, Belgium |
| Projection | EPSG:32631 — WGS 84 / UTM zone 31N |
| Bounding box (UTM) | Easting 610,000 → 650,030 m · Northing 5,571,802 → 5,602,572 m |
| Imagery | Sentinel-2 L2A, 2023, cloud cover < 20% |

---

## How Compute and Storage Work in This Project

This is one of the key cloud concepts the project teaches.

**Azure Blob Storage is just an API.** It doesn't care where your code runs — your laptop, a cloud VM, or anything else. As long as you have the connection string in `config.py`, your notebook reads and writes to the same data from anywhere.

```
Your laptop                 Azure ML Compute Instance
      │                              │
      └──────────────┬───────────────┘
                     │
              Azure Blob Storage
            (shared data for the group)
```

This means:
- You can start work on your laptop and continue on the cloud — the data is always in Blob Storage
- Your group shares one Blob Storage container — everyone reads raw data and writes their own results
- The notebook code is **identical** whether running locally or on the cloud instance

**When to use which environment:**

| Task | Local laptop | Cloud instance (E4ds_v4) |
|---|---|---|
| Writing and testing code | ✅ Ideal | ✅ Works |
| Loading a single band (~116 MB) | ✅ Fine | ✅ Fine |
| Monthly compositing | ✅ Fine | ✅ Fine |
| Full feature stack in memory (~8 GB) | ⚠️ Needs 16 GB+ RAM | ✅ 32 GB — no problem |
| RF training on training pixels | ✅ Fine | ✅ Fine |
| Predicting 124M pixels (~20 GB) | ❌ Will fail on most laptops | ✅ 32 GB — no problem |
| Visualisation and results | ✅ Ideal | ✅ Works |

> 💡 **Simple rule:** write and explore locally, run the heavy steps on the cloud.
> If your laptop has less than 16 GB RAM, just use the cloud instance for everything — same code, no changes needed.

---

## Files Provided by Your Trainer

| File | What it is | Where to put it |
|---|---|---|
| `config.py` | Pre-filled configuration — you add your `MY_PREFIX` only | Both laptop and JupyterLab |
| `wallonia_insitu_2023.shp` + companions | Merged LPIS 2023 + Walous 2023 training polygons | JupyterLab (Day 3) |
| `crop_dictionary.csv` | Numeric class codes mapped to crop/land cover names | JupyterLab (Day 4) |
| `test_auth.py` | Tests Copernicus authentication — just run it | Both |
| `test_search.py` | Verifies STAC search returns 39 scenes — just run it | Both |
| `download_s2.py` | Downloads Sentinel-2 data to Blob Storage — just run it | Both |
| `student_workbook.ipynb` | Your coding workbook for Days 1–4 | Both |

---

## Sentinel-2 Bands

| Band | Wavelength | Resolution | Role |
|---|---|---|---|
| B02 — Blue | 490 nm | 10 m | Water, bare soil, urban |
| B03 — Green | 560 nm | 10 m | Vegetation green peak |
| B04 — Red | 665 nm | 10 m | Chlorophyll, NDVI |
| B05 — Red Edge 1 | 705 nm | 20 m | Early crop stress |
| B06 — Red Edge 2 | 740 nm | 20 m | Canopy chlorophyll |
| B07 — Red Edge 3 | 783 nm | 20 m | Leaf area index |
| B08 — NIR broad | 842 nm | 10 m | Vegetation vigour |
| B8A — NIR narrow | 865 nm | 20 m | Canopy structure |
| B11 — SWIR-1 | 1610 nm | 20 m | Soil moisture, crop stress |
| B12 — SWIR-2 | 2190 nm | 20 m | Crop type discrimination |
| SCL | — | 20 m | Cloud mask — not a classification feature |

---

# DAY 1 — Morning · Individual Azure Setup

> **Everyone does this independently.** The goal is hands-on cloud infrastructure experience.
> You will keep your own Azure account and compute instance for the whole project.

---

## Step 1 — Activate Azure for Students

1. Open **https://azure.microsoft.com/en-us/free/students**
2. Click **Start free**
3. Sign in using your **BeCode `.edu` email**
4. Complete student identity verification — no credit card required
5. Confirm **$100 credit** appears in your account

> ⚠️ **BeCode tenant restrictions**
> Your account may block resource creation in certain Azure regions.
> If you see a 403 error or *"region disallowed"* message, try regions in this order:
> **Belgium Central → Sweden Central**
> Use the same region for all resources you create.

---

## Step 2 — Create a Resource Group

1. Go to **portal.azure.com**
2. Search **Resource groups** → **+ Create**
3. Fill in:
   - **Subscription:** Azure for Students
   - **Resource group name:** `my-crop-rg`
   - **Region:** Belgium Central *(or Sweden Central if blocked)*
4. Click **Review + Create** → **Create**

---

## Step 3 — Assign Yourself the Required Roles

1. Search **Subscriptions** → click **Azure for Students**
2. Left menu → **Access control (IAM)** → **+ Add** → **Add role assignment**
3. Search **Storage Account Contributor** → select → **Next**
4. **Members** → **+ Select members** → search your email → select → **Review + assign**
5. Repeat for **Storage Blob Data Contributor**

> 📝 Wait ~60 seconds after assigning roles before creating the storage account.
> If you see a **Contributor** role in the list, assign that instead — it covers everything.

---

## Step 4 — Create a Storage Account

1. Search **Storage accounts** → **+ Create**
2. Fill in:

   | Field | Value |
   |---|---|
   | Resource group | `my-crop-rg` |
   | Storage account name | `mycropdata` + your initials — globally unique, lowercase, max 24 chars, no hyphens |
   | Region | Same as your resource group |
   | Primary workload | Other |
   | Performance | Standard |
   | Redundancy | LRS |

3. Click through all other tabs without changing anything
4. **Review + Create** → **Create**

Once deployed:

5. Left menu → **Containers** → **+ Container** → Name: `sentinel2-data` · Private → **Create**
6. Left menu → **Access keys** → **Show** key1 → **copy it** and save somewhere safe

> ⚠️ If you get *"Resource was disallowed"* — try the next region in: Belgium Central → Sweden Central.

---

## Step 5 — Create an Azure ML Workspace

1. Search **Azure Machine Learning** → **+ Create** → **New workspace**
2. Fill in:
   - **Resource group:** `my-crop-rg`
   - **Name:** `my-crop-ws`
   - **Region:** same as storage account
3. Leave all defaults → **Review + Create** → **Create** *(~2 minutes)*
4. Click **Launch Studio** → opens **ml.azure.com**

---

## Step 6 — Create a Compute Instance

1. Azure ML Studio → **Compute** → **Compute Instances** → **+ New**
2. Fill in:
   - **Name:** `my-compute`
   - **VM size:** `Standard_E4ds_v4` *(4 cores, 32 GB RAM — required for the heavy pipeline steps)*
3. Click **Create** → wait ~3 minutes for **Running** status
4. Click **JupyterLab** to open your notebook environment

> 💡 While waiting: explore the Azure ML Studio interface and notice the Cost Management section.

---

## Step 7 — Set Up Your Local Environment

Even if you plan to work mostly on the cloud instance, set up your local environment now. You will use it for writing and testing code and for light tasks that don't need the cloud VM.

Open a terminal on your laptop:

```bash
# Create a project folder
mkdir crop-classification
cd crop-classification

# Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate          # Mac/Linux
# .venv\Scripts\activate           # Windows

# Install packages
pip install rasterio geopandas azure-storage-blob scikit-learn
pip install matplotlib scipy pandas numpy shapely pystac-client requests boto3
```

> 📝 **Windows users:** if `rasterio` fails to install, use:
> `pip install rasterio --find-links https://girder.github.io/large_image_wheels`

---

## Step 8 — Set Up Notebook Sync with Git

To move your notebook between your laptop and the cloud instance without manual file copying, use Git.

**On your laptop:**
```bash
cd crop-classification
git init
git add .
# Create a repo on GitHub/GitLab and push to it
git remote add origin https://github.com/yourusername/crop-classification.git
git push -u origin main
```

**On your cloud instance** (in JupyterLab Terminal):
```bash
git clone https://github.com/yourusername/crop-classification.git
cd crop-classification
```

From then on, syncing is:
```bash
# Save progress from laptop to cloud
git add . && git commit -m "day2 progress" && git push

# Pull latest on cloud instance
git pull
```

> 💡 Commit `config.py` to Git only if your repo is **private**. If public, add `config.py` to `.gitignore` and share it through another channel — it contains your storage key.

---

## Step 9 — Install Packages on Cloud Instance and Verify

In JupyterLab Terminal:

```bash
pip install rasterio geopandas azure-storage-blob scikit-learn
pip install matplotlib scipy pandas numpy shapely pystac-client requests boto3
```

Upload `config.py` to JupyterLab (drag and drop). Open it and add your `STORAGE_KEY`.

📓 **Notebook — Step 1:** Load config and verify all variables print correctly

> ✅ Morning checkpoint:
> - Running cloud compute instance (JupyterLab accessible)
> - Local environment set up and packages installed
> - Git repo created and cloned on both environments
> - `config.py` filled in with your storage key

> ⚠️ **Do NOT stop your compute instance yet — you need it for the afternoon.**

---

# DAY 1 — Afternoon · Group Setup & Data Download

---

## Part A — Nominate a Cloud Lead and Set Up Shared Storage

Each group of 3–4 nominates one **Cloud Lead**. The Cloud Lead's Blob Storage becomes the group's shared data store for the whole project.

**Why shared storage?**
- Raw Sentinel-2 data (~50 GB) only needs to be downloaded once
- All group members read from the same raw data — no duplication
- Each person writes their own composites and results to their own storage account

**Group roles:**

| Role | Responsibilities |
|---|---|
| Cloud Lead | Runs the data download. Shares storage connection string with teammates. Monitors group cost. |
| Data Engineer | Day 2: leads cloud masking, compositing, feature stack. |
| ML Engineer (1–2 people) | Day 3–4: leads feature selection, training, prediction, visualisation. |

> 📝 Roles are not rigid — everyone writes and runs code. The roles just define who leads each day.

---

## Part B — Cloud Lead: Share Credentials with Teammates

The Cloud Lead shares three things with the group — verbally or in a private group chat:

1. **Storage account name** (e.g. `mycroptamt`)
2. **key1** — from portal.azure.com → your storage account → **Access keys** → Show → copy key1
3. **Container name** (`sentinel2-data`)

Your trainer will pre-fill `config.py` with all of these. Everyone in the group uses the **same** `STORAGE_ACCOUNT`, `STORAGE_KEY`, and `CONTAINER_NAME`.

**The only thing each person sets themselves is `MY_PREFIX`** — your personal folder name inside the shared container. This prevents anyone from accidentally overwriting a teammate's files.

```python
# config.py — everyone uses the Cloud Lead's storage credentials
STORAGE_ACCOUNT = "cloud_leads_storage_name"   # same for everyone in the group
STORAGE_KEY     = "cloud_leads_key1"            # same for everyone in the group
CONTAINER_NAME  = "sentinel2-data"             # same for everyone in the group

MY_PREFIX       = "alice"    # ← each person sets their own name here
```

This means your files in Blob Storage are organised like:

```
sentinel2-data/
├── raw/                          ← shared raw S2 data (Cloud Lead downloaded)
│   └── T31UFS/
│       ├── 20230601_B04_10m.tif
│       └── ...
├── composites/
│   ├── alice/                    ← Alice's composites
│   │   ├── 202306_B04_10m.tif
│   │   └── ...
│   ├── bob/                      ← Bob's composites
│   └── ...
└── results/
    ├── alice/                    ← Alice's classification maps
    └── bob/
```

Everyone reads from the same `raw/` folder but writes to their own prefix. No collisions, no overwriting.

> ⚠️ Keep the storage key within your group. Do not post it publicly or commit it to a public Git repository.

---

## Part C — Set a Budget Alert (Everyone)

Each person sets a budget alert on their own subscription:

1. Search **Cost Management** → **Budgets** → **+ Add**
2. Name: `bootcamp-budget` · Amount: **$20**
3. Alert at 80% ($16) · add your email
4. Click **Create**

**Expected cost per person over 4 days:**

| Resource | Cost |
|---|---|
| E4ds_v4 compute instance (6h/day × 4 days, stopped nightly) | ~$7 |
| Blob Storage (~15 GB composites + results) | < $0.03 |
| Reading Cloud Lead's raw data | < $0.10 |
| **Total** | **< $8** |

---

## Part D — Run the Sentinel-2 Download (Cloud Lead only)

The Cloud Lead runs the download. **Teammates do not need to download — they will read from the shared storage.**

Upload `download_s2.py` to JupyterLab and run it in a Terminal:

```bash
python3 download_s2.py
```

**What to expect:**
- 39 scenes × 11 bands = **429 files** · ~116 MB each · ~50 GB total
- Estimated time: **2–4 hours** — start it now and let it run
- Safe to interrupt and restart — already-uploaded files are skipped

> ⚠️ Cloud Lead: while the download runs, continue with the notebook steps below.
> The download runs independently in the Terminal — your notebook is unaffected.

📓 **Notebook — Steps 2–4 (everyone):** While the download runs, implement the authentication, STAC search, and Blob Storage helper functions. These work the same whether you run them locally or on the cloud instance.

---

## Part E — Fallback: Use Trainer's Pre-Downloaded Data

If the download fails, your trainer has pre-staged all data. A single cell in the notebook copies it server-side in minutes.

📓 **Notebook — Step 4:** Fallback copy cell — run this if needed

---

## Part F — Configure Your Notebook

Upload `config.py` to JupyterLab *(drag and drop into the file browser)* and also place a copy in your local project folder.

Your trainer has pre-filled everything. **The only thing you set yourself is `MY_PREFIX`** — use your first name or initials.

| Variable | Filled by | Value |
|---|---|---|
| `STORAGE_ACCOUNT` | Trainer | Cloud Lead's storage account name |
| `STORAGE_KEY` | Trainer | Cloud Lead's key1 |
| `CONTAINER_NAME` | Trainer | `sentinel2-data` |
| **`MY_PREFIX`** | **You** | **Your first name or initials e.g. `alice`** |
| `S3_ACCESS_KEY` | Trainer | Shared Copernicus S3 key |
| `S3_SECRET_KEY` | Trainer | Shared Copernicus S3 secret |
| `BACKUP_SAS_TOKEN` | Trainer (verbally) | Read-only token to trainer's pre-downloaded data |
| `ROI_BBOX` | Trainer | `[4.55, 50.25, 5.15, 50.55]` |
| `BANDS` | Trainer | All 11 band asset keys |

> 💡 `MY_PREFIX` is the only variable that differs between group members.
> Every blob path in your notebook uses it — `f"composites/{MY_PREFIX}/..."` — so your files never overwrite a teammate's.

> ⚠️ Do not commit `config.py` to a **public** Git repository — it contains your group's storage key.
> Add it to `.gitignore` if your repo is public, and share it through another channel.

📓 **Notebook — Step 1:** Load config, verify `MY_PREFIX` prints correctly, list files in Blob Storage

---

## Switching Between Local and Cloud During the Project

From Day 2 onwards, work wherever makes sense. The workflow is always the same:

**Moving from laptop → cloud:**
```bash
# On laptop: save and push
git add . && git commit -m "describe what you did" && git push

# On cloud instance (JupyterLab Terminal): pull latest
git pull
```

**Moving from cloud → laptop:**
```bash
# On cloud instance (JupyterLab Terminal): push
git add . && git commit -m "describe what you did" && git push

# On laptop: pull
git pull
```

> 💡 The data always stays in Blob Storage — you never need to move it. Only the notebook file moves via Git.

---

## End of Day 1 Checklist

- ✅ Azure account activated, resource group + storage + ML workspace + compute instance created
- ✅ Local environment set up with packages installed
- ✅ Git repo created and synced between laptop and cloud
- ✅ `config.py` filled in with your `MY_PREFIX` — blob listing works from both environments
- ✅ Cloud Lead running data download (or fallback copy triggered)
- ✅ Budget alert set at $20

> ⚠️ **STOP your compute instance before leaving:**
> Azure ML Studio → Compute → your instance → **Stop**
> Closing the browser does NOT stop billing — you must explicitly click Stop.

---

# DAY 2 — Cloud Masking, Compositing & Feature Stack

> **Data Engineer leads today.** Everyone follows along in their own notebook.
> Write and test your functions locally first, then run the heavy steps on the cloud instance.

**What you are building:**
- Monthly median composites for all 10 feature bands (cloud-masked)
- NDVI, NDWI, NDBI spectral indices per month
- A 3D feature array `(rows, cols, n_features)` — your input to the classifier

---

## Overview: Cloud Masking with SCL

The SCL (Scene Classification Layer) labels each pixel. Keep only valid pixels:

| SCL value | Meaning | Keep? |
|---|---|---|
| 4 | Vegetation | ✅ |
| 5 | Non-vegetated | ✅ |
| 6 | Water | ✅ |
| 7 | Unclassified | ✅ |
| 11 | Snow / Ice | ✅ |
| All others | Cloud, shadow, saturated | ❌ |

## Overview: Monthly Median Composite

For each month, take the pixel-wise **median** across all cloud-free observations. The median is more robust than the mean for handling residual cloud contamination.

## Overview: Spectral Indices

| Index | Formula | Captures |
|---|---|---|
| NDVI | (B08 − B04) / (B08 + B04) | Vegetation density and phenology |
| NDWI | (B03 − B08) / (B03 + B08) | Water bodies, irrigation |
| NDBI | (B11 − B08) / (B11 + B08) | Built-up surfaces |

---

📓 **Notebook — Steps 5–9**

| Step | Task | Where to run |
|---|---|---|
| Step 5 | Group scenes by month | Local or cloud |
| Step 6 | Cloud masking function | Local or cloud |
| Step 7 | Monthly median composites | Local or cloud — one band at a time is fine |
| Step 8 | Spectral indices | Local or cloud |
| Step 9 | Assemble full feature stack | **Cloud recommended** if laptop < 16 GB RAM |

---

## End of Day 2 Checklist

- ✅ Monthly composites saved for all 10 bands in `composites/<MY_PREFIX>/` in Blob Storage
- ✅ NDVI, NDWI, NDBI composites saved (36 files under your prefix)
- ✅ Feature stack shape confirmed e.g. `(3100, 4000, 156)`
- ✅ `feature_names.json` saved — push to Git so it's available on both environments

> ⚠️ **STOP your compute instance before leaving!**

---

# DAY 3 — Feature Selection & Classification

> **ML Engineer leads today.**
> Feature selection is a group discussion — examine importance scores together before deciding.
> **Run all of Day 3 on the cloud instance** — feature stack and prediction need 32 GB RAM.

**What you are building:**
- A rasterized training map from LPIS + Walous polygons
- A feature importance ranking and selected subset
- A trained Random Forest classifier
- A full prediction map saved to Blob Storage

---

## Overview: Training Data

Your trainer's shapefile combines:
- **LPIS 2023** — officially declared crop types for each agricultural parcel in Wallonia
- **Walous 2023** — non-agricultural classes (forest, urban, water, bare soil)

Each labeled polygon is rasterized onto the same pixel grid as your feature stack. Each labeled pixel becomes one training sample.

---

## Overview: Feature Selection

With up to 156 features, not all are equally useful. You will train a quick Random Forest on a subsample to rank features by **Gini importance**, then choose a subset.

**Group discussion — answer these before selecting your subset:**
- Which months rank highest? Does this make agronomic sense for Wallonia?
- Are NDVI, NDWI, NDBI in the top features?
- Where does importance drop off sharply?
- How does your accuracy change between 20, 40, and 80 features?

---

## Overview: Random Forest

Key parameters:
- `n_estimators=100` — 100 trees, majority vote for predictions
- `oob_score=True` — free accuracy estimate on out-of-bag samples
- `n_jobs=-1` — uses all 4 CPU cores

**OOB accuracy target: > 80%.** If lower, check your training data and feature selection.

---

📓 **Notebook — Steps 10–15 (run on cloud instance)**

| Step | Task |
|---|---|
| Step 10 | Reload feature stack from Blob Storage |
| Step 11 | Load and rasterize the training shapefile |
| Step 12 | Build X/y matrices and 80/20 train/test split |
| Step 13 | Rank features by importance, plot, select subset |
| Step 14 | Train the Random Forest |
| Step 15 | Predict the full image and save to Blob Storage |

---

## End of Day 3 Checklist

- ✅ Training data rasterized with CRS verified
- ✅ Feature importance explored and subset choice documented
- ✅ Random Forest trained — OOB accuracy noted
- ✅ Full prediction map saved to `results/<MY_PREFIX>/classification_RF.tif`
- ✅ Push notebook to Git

> ⚠️ **STOP your compute instance before leaving!**

---

# DAY 4 — Evaluation, Post-Processing & Presentation

> Day 4 is mostly light processing and visualisation — **local laptop is fine** for everything except loading the full feature stack if needed.

**What you are building:**
- Accuracy report on held-out test set
- Reclassified map using crop dictionary
- Spatially smoothed final map
- Publication-quality visualisation
- Your group presentation

---

## Overview: Accuracy Assessment

You reserved 20% of training pixels as a held-out test set — never seen during training.

**Report:**
- **Overall Accuracy (OA):** percentage of correctly classified pixels
- **Per-class Precision, Recall, F1-score**

**Reflection questions:**
- Which classes have the lowest F1-score? Why?
- How does OA compare to OOB from training?
- High precision + low recall for a class — what does that mean in practice?

---

## Overview: Reclassification

The crop dictionary maps detailed LPIS codes to broader readable groups — e.g. `Winter wheat`, `Spring wheat`, `Durum wheat` → `Cereals`. This makes the final map more interpretable.

---

## Overview: Majority Filter

Replaces each pixel with the most common class in its 3×3 neighbourhood. Removes salt-and-pepper noise without significantly changing overall accuracy.

---

📓 **Notebook — Steps 16–20**

| Step | Task | Where to run |
|---|---|---|
| Step 16 | Accuracy assessment | Local or cloud |
| Step 17 | Reclassify with crop dictionary | Local or cloud |
| Step 18 | Majority filter | Local or cloud |
| Step 19 | Visualise final map | Local — easier to iterate on figures |
| Step 20 | Download results to laptop | Local |

---

## Presentation Guide (15 minutes per group)

1. **Infrastructure** — which Azure services did you use? What surprised you about cloud setup?
2. **Data** — how many scenes? What cloud cover patterns did you see across 2023?
3. **Feature selection** — which features ranked highest? Did this match expectations?
4. **Results** — show your map. Overall accuracy? Hardest classes to classify?
5. **Reflection** — what would you do differently with more time or budget?

---

## Step 21 — Clean Up Azure Resources

> ⚠️ **Download all results before doing this.**

1. **portal.azure.com** → **Resource groups** → your resource group
2. **Delete resource group** → type name to confirm → **Delete**

This removes everything and stops all costs. Your $100 credit balance is unaffected.

---

## End of Day 4 Checklist

- ✅ Accuracy report saved
- ✅ Reclassified and filtered map in Blob Storage
- ✅ Final map exported as PNG
- ✅ Results downloaded to laptop
- ✅ Presentation delivered
- ✅ Resource group deleted

---

# Quick Reference

## Cost Summary (per person, 4 days)

| Resource | Cost |
|---|---|
| E4ds_v4 compute instance, stopped nightly | ~$7 |
| Blob Storage ~15 GB | < $0.03 |
| Reading from shared raw data | < $0.10 |
| **Total** | **< $8** |

> ⚠️ Leaving E4ds_v4 running overnight (14h) costs ~$4.20. Four nights = ~$17.
> **Always click Stop — not just close the browser tab.**

## VM Sizes Reference

| VM | RAM | Cost/hr | Use for |
|---|---|---|---|
| Standard_DS2_v2 | 7 GB | ~$0.14 | Light tasks only |
| Standard_DS3_v2 | 14 GB | ~$0.25 | Compositing, small datasets |
| **Standard_E4ds_v4** | **32 GB** | **~$0.30** | **This project — recommended** |
| Standard_DS4_v2 | 28 GB | ~$0.50 | Alternative to E4ds_v4 |

## Useful Links

| Resource | URL |
|---|---|
| Azure for Students | https://azure.microsoft.com/en-us/free/students |
| Azure Portal | https://portal.azure.com |
| Azure ML Studio | https://ml.azure.com |
| Copernicus Data Space | https://dataspace.copernicus.eu |
| Copernicus S3 credentials | https://eodata-s3keysmanager.dataspace.copernicus.eu |
| QGIS (free GIS viewer) | https://qgis.org |
| Reference classification notebook | https://nicolasdeffense.github.io/eo-toolbox/notebooks/7_Classification/random_forest_classification.html |

## SCL Cloud Mask Values

| SCL | Meaning | Keep? |
|---|---|---|
| 0 | No data | ❌ |
| 1 | Saturated / defective | ❌ |
| 2 | Dark area | ❌ |
| 3 | Cloud shadows | ❌ |
| 4 | Vegetation | ✅ |
| 5 | Non-vegetated | ✅ |
| 6 | Water | ✅ |
| 7 | Unclassified | ✅ |
| 8 | Cloud medium probability | ❌ |
| 9 | Cloud high probability | ❌ |
| 10 | Thin cirrus | ❌ |
| 11 | Snow / Ice | ✅ |
