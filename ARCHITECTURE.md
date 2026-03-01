# Pannaverse Architecture

Comprehensive system architecture for the pannaverse ecosystem. For pipeline-level details, see [`panna/CLAUDE.md`](panna/CLAUDE.md). For data structure details, see [`pannadata/README.md`](pannadata/README.md).

## 1. System Overview

Pannaverse is a football (soccer) player rating system built on **RAPM+SPM methodology**. Raw match data flows from three sources (Opta, FBref, Understat) through automated scrapers into GitHub Releases, where R pipelines consume it to produce player ratings and match predictions. The final output reaches an end-user blog via Cloudflare R2.

### Key Terminology

| Term | Definition |
|------|-----------|
| **RAPM** | Regularized Adjusted Plus-Minus — lineup-based player impact from ridge regression on splint-level data |
| **SPM** | Statistical Plus-Minus — box score prediction of RAPM using elastic net on per-90 stats |
| **xRAPM** | RAPM re-fit with SPM as a Bayesian prior for stability |
| **Panna Rating** | Final combined rating: `offense - defense` (higher = better) |
| **Splint** | A time segment where the lineup is constant (boundaries at goals, subs, red cards, halftime) |
| **SPADL** | Soccer Player Action Description Language — standardized event format used for EPV |
| **EPV** | Expected Possession Value — action-level player valuation |
| **xMetrics** | Pre-computed xG, xA, and xPass per player per match from SPADL + XGBoost models |
| **Estimated Skills** | Bayesian decay-weighted skill estimates with position-specific priors |

## 2. Full Data Flow

```
                          ┌─────────────┐
                          │ Data Sources │
                          └──────┬──────┘
                ┌────────────────┼────────────────┐
                ▼                ▼                ▼
          ┌──────────┐    ┌──────────┐    ┌────────────┐
          │   Opta   │    │  FBref   │    │ Understat  │
          │ (Python) │    │   (R)    │    │    (R)     │
          │  GHA 5AM │    │ VM  6AM  │    │  GHA 7AM   │
          └────┬─────┘    └────┬─────┘    └─────┬──────┘
               │               │                │
               ▼               ▼                ▼
        ┌────────────┐  ┌────────────┐   ┌────────────┐
        │opta-latest │  │fbref-latest│   │understat-  │
        │  (release) │  │  (release) │   │latest (rel)│
        └─────┬──────┘  └─────┬──────┘   └────────────┘
              │               │
    ┌─────────┼───────────────┘
    │         │
    ▼         ▼
┌────────────────────┐     ┌────────────────────────┐
│ Opta RAPM/SPM/     │     │ FBref RAPM/SPM Pipeline │
│ Skills Pipelines   │     │  (GHA, manual dispatch) │
│ (local + GHA)      │     └───────────┬────────────┘
└────────┬───────────┘                 │
         │                             ▼
         │                      ┌─────────────┐
         │                      │ ratings-data │
         │                      │  (release)   │
         │                      └─────────────┘
         ▼
  ┌───────────────┐    ┌───────────────────┐
  │ predictions-  │    │  predictions-     │
  │ cache (rel)   │◄───│  cache upload     │
  └───────┬───────┘    │  (manual script)  │
          │            └───────────────────┘
          ▼
  ┌───────────────────┐
  │ Predictions       │
  │ Pipeline          │
  │ (GHA, Wed 8AM)    │
  └───────┬───────────┘
          │
    ┌─────┴──────┐
    ▼            ▼
┌──────────┐ ┌──────────┐
│predict-  │ │blog-     │
│ions-     │ │latest    │
│latest    │ │(release) │
└──────────┘ └────┬─────┘
                  │  repository_dispatch
                  ▼
          ┌───────────────┐
          │build-blog-data│
          │  (GHA)        │
          └───────┬───────┘
                  │  wrangler r2
                  ▼
          ┌───────────────┐
          │ Cloudflare R2 │
          │ inthegame-data│
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │inthegame-blog │
          └───────────────┘
```

## 3. Orchestration Timeline

### Daily (automated)

| Time (UTC) | Workflow | Repo | Runs On | What It Does |
|------------|----------|------|---------|-------------|
| 5:00 AM | `daily-opta-scrape.yml` | pannadata | GHA | Scrapes 15 Opta leagues → uploads consolidated parquets to `opta-latest` release |
| 6:00 AM | `scrape_fbref.R` (cron) | pannadata | Oracle VM | Scrapes FBref → uploads to `fbref-latest` release |
| 7:00 AM | `daily-understat-scrape.yml` | pannadata | GHA | Scrapes 6 Understat leagues → uploads to `understat-latest` release |

### Weekly (automated)

| Time (UTC) | Workflow | Repo | Runs On | What It Does |
|------------|----------|------|---------|-------------|
| Wed 8:00 AM | `predictions-pipeline.yml` | panna | GHA | Downloads Opta data + caches → runs prediction steps 01-10 → uploads to `predictions-latest` and `blog-latest` → triggers `build-blog-data.yml` |

### On-demand (triggered)

| Trigger | Workflow | Repo | What It Does |
|---------|----------|------|-------------|
| `repository_dispatch: predictions-complete` | `build-blog-data.yml` | pannadata | Downloads from `blog-latest` → pushes parquets to Cloudflare R2 |
| `repository_dispatch: opta-scrape-complete` | `predictions-pipeline.yml` | panna | Runs predictions pipeline (same as weekly) |
| `repository_dispatch: scrape-complete` | `scrape-notification.yml` | pannadata | Logs FBref scrape summary to GHA step summary |
| Manual dispatch | `ratings-pipeline.yml` | panna | Runs FBref RAPM/SPM pipeline → uploads to `ratings-data` release |
| Manual dispatch | `build-blog-data.yml` | pannadata | Same as triggered version |

### Manual (local)

| Task | Script | What It Does |
|------|--------|-------------|
| Opta RAPM/SPM pipeline | `panna/data-raw/player-ratings-opta/run_pipeline_opta.R` | Full 15-league RAPM/SPM/Panna rating pipeline (~36 min) |
| Skills pipeline | `panna/data-raw/estimated-skills/run_skills_pipeline.R` | Bayesian skill estimation with 3-pass optimization |
| xMetrics pipeline | `panna/data-raw/epv/03_calculate_player_xmetrics.R` | Computes xG/xA/xPass for all leagues |
| Upload caches | `panna/data-raw/match-predictions-opta/upload_prediction_caches.R` | Pushes RAPM + skills caches to `predictions-cache` release |

## 4. GitHub Releases as Data Bus

GitHub Releases serve as the data transport layer between repos and workflows. All releases live on `peteowen1/pannadata`.

| Release Tag | Contents | Written By | Read By |
|-------------|----------|-----------|---------|
| `opta-latest` | 9 consolidated parquets + manifest + catalog + per-league events | `daily-opta-scrape.yml` | `predictions-pipeline.yml`, local pipelines via `pb_download_opta()` |
| `fbref-latest` | FBref parquet archives | Oracle VM `scrape_fbref.R` | `ratings-pipeline.yml`, local pipelines via `pb_download_source("fbref")` |
| `understat-latest` | Consolidated Understat parquets + manifest | `daily-understat-scrape.yml` | Local pipelines via `pb_download_source("understat")` |
| `epv-models` | `xg_model.rds`, `xpass_model.rds`, `epv_model.rds` | Manual (one-time) | xMetrics pipeline via `pb_download_epv_models()` |
| `predictions-cache` | 5 `.rds` cache files (RAPM + skills) | `upload_prediction_caches.R` (manual) | `predictions-pipeline.yml` |
| `predictions-latest` | `predictions.parquet`, `predictions.csv` | `predictions-pipeline.yml` step 09 | Archive / analysis |
| `blog-latest` | `panna_ratings.parquet`, `match_predictions.parquet` | `predictions-pipeline.yml` step 10 | `build-blog-data.yml` |
| `ratings-data` | `seasonal_xrapm.parquet`, `seasonal_spm.parquet` | `ratings-pipeline.yml` | `build_blog_data.R` (alternate path) |

### Predictions-cache contents

| File | Source Pipeline |
|------|---------------|
| `cache-opta/07_seasonal_ratings.rds` | Opta RAPM pipeline |
| `cache-skills/06_seasonal_ratings.rds` | Skills pipeline |
| `cache-skills/01_match_stats.rds` | Skills pipeline |
| `cache-skills/02b_decay_params.rds` | Skills pipeline |
| `cache-skills/03_skill_spm.rds` | Skills pipeline |

## 5. Pipeline Dependency Graph

```
EPV models (one-time train)
    │
    ▼
xMetrics pipeline ──────────────────────┐
    │                                   │
    ▼                                   ▼
Opta RAPM/SPM pipeline          Skills pipeline
    │                              │
    │    ┌─────────────────────────┘
    │    │
    ▼    ▼
upload_prediction_caches.R (manual)
    │
    ▼
predictions-cache release
    │
    ▼
Predictions pipeline (GHA) ◄──── Opta daily scrape
    │
    ├──→ predictions-latest release
    └──→ blog-latest release
              │
              ▼
         build-blog-data.yml
              │
              ▼
         Cloudflare R2 → inthegame-blog

FBref scrape (Oracle VM) ──→ FBref RAPM/SPM pipeline (GHA)
                                    │
                                    ▼
                              ratings-data release
```

### What feeds what

- **xMetrics** requires pre-trained EPV models (`epv-models` release)
- **Opta RAPM/SPM** requires xMetrics output (pre-generated parquets in `pannadata/data/opta/xmetrics/`)
- **Skills pipeline** requires Opta RAPM output (`cache-opta/`) and xMetrics
- **Predictions pipeline** requires Opta daily data (`opta-latest`) + uploaded caches (`predictions-cache`)
- **Blog** requires predictions pipeline output (`blog-latest`)
- **FBref pipeline** is independent — uses FBref data only, produces `ratings-data`

## 6. Cache Lifecycle

### Local caches (gitignored)

| Directory | Contents | Produced By | Consumed By |
|-----------|----------|------------|-------------|
| `panna/data-raw/cache-opta/` | RAPM pipeline intermediate + final outputs (01-09 `.rds` files) | Opta RAPM pipeline | Skills pipeline, predictions (via upload) |
| `panna/data-raw/cache-skills/` | Skills pipeline outputs (01-06 `.rds` files) | Skills pipeline | Predictions (via upload) |
| `panna/data-raw/cache-predictions-opta/` | Prediction pipeline intermediates + final | Predictions pipeline | Blog export (step 10) |
| `panna/data-raw/cache/` | FBref RAPM pipeline caches | FBref pipeline | FBref pipeline only |

### When to refresh caches

| Scenario | Action |
|----------|--------|
| New season data or model changes | Re-run Opta RAPM pipeline locally → re-run Skills pipeline → run `upload_prediction_caches.R` |
| Skills methodology changes | Re-run Skills pipeline → run `upload_prediction_caches.R` |
| Prediction model changes | Re-trigger `predictions-pipeline.yml` (GHA) or run locally |
| EPV model retraining | Run `01_train_epv_models.R` → upload to `epv-models` release → re-run xMetrics |

### Upload caches to GitHub Releases

```bash
cd panna
Rscript data-raw/match-predictions-opta/upload_prediction_caches.R
```

This uploads the 5 cache files to the `predictions-cache` release so the GHA predictions pipeline can access them.

## 7. Cloudflare R2 Integration

| Item | Value |
|------|-------|
| **Bucket** | `inthegame-data` |
| **Upload tool** | Cloudflare Wrangler CLI (`wrangler r2 object put`) |
| **Files stored** | `panna_ratings.parquet`, `match_predictions.parquet` |
| **Written by** | `build-blog-data.yml` workflow |
| **Read by** | `inthegame-blog` (blog website) |
| **Auth secrets** | `CLOUDFLARE_R2_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` (in pannadata repo secrets) |

### Data flow

```
predictions-pipeline.yml (panna)
    → uploads panna_ratings.parquet + match_predictions.parquet to blog-latest release
    → sends repository_dispatch "predictions-complete" to pannadata

build-blog-data.yml (pannadata)
    → downloads both parquets from blog-latest release
    → wrangler r2 object put "inthegame-data/<filename>" --file "<file>" --remote

inthegame-blog
    → reads parquets from inthegame-data R2 bucket
```

### Alternate path (standalone R script)

`pannadata/scripts/build_blog_data.R` can build `panna_ratings.parquet` from `ratings-data` release files. This is an alternate path that doesn't require the predictions pipeline — it reads `seasonal_xrapm.parquet`, `seasonal_spm.parquet`, and `player_metadata.parquet` directly.

## 8. Environment Constraints

| Component | Runs On | Why |
|-----------|---------|-----|
| Opta scraper | GitHub Actions | Python-based, no IP restrictions |
| Understat scraper | GitHub Actions | R-based, no IP restrictions |
| FBref scraper | Oracle Cloud VM | **FBref blocks GitHub Actions IP addresses** |
| Opta RAPM/SPM pipeline | Local machine | Too heavy for GHA (~36 min, large memory) |
| Skills pipeline | Local machine | Requires RAPM output, parallelized optimization |
| xMetrics pipeline | Local machine | Requires pre-trained EPV models + SPADL conversion |
| FBref RAPM pipeline | GitHub Actions | Lighter than Opta pipeline (Big 5 + UCL/UEL only) |
| Predictions pipeline | GitHub Actions | Lightweight — uses pre-computed caches |

### Oracle VM details

- **Address**: `opc@168.138.108.69`
- **Cron**: FBref scrape at 6 AM UTC daily
- **Post-scrape**: Sends `repository_dispatch: scrape-complete` to pannadata (logged by `scrape-notification.yml`)
- **Data upload**: `--upload` flag pushes parquets to `fbref-latest` release

### GHA R version requirement

Predictions and ratings pipelines require **R 4.5.0+** (`ggrepel` dependency).

### Cross-repo auth

All cross-repo GHA operations use `WORKFLOW_PAT` secret (personal access token with `repo` scope), stored in the `panna` repo.

## 9. Failure Modes & Recovery

| Failure | Cause | Recovery |
|---------|-------|----------|
| Opta scrape OOM | Large consolidated parquet in memory | Fixed by streaming approach (PR #16). If recurs, increase `tier` input to scrape fewer leagues |
| FBref scrape fails | IP block or site changes | Run from Oracle VM. Check `scrape-notification.yml` step summary for error details |
| FBref GHA IP blocked | FBref blocks GHA IP ranges | Use Oracle VM — `daily-fbref-scrape.yml` is disabled for this reason |
| Predictions pipeline fails | Missing caches or stale data | Ensure `predictions-cache` release has current caches. Run `upload_prediction_caches.R` after any RAPM/Skills pipeline changes |
| Blog data not updating | `build-blog-data.yml` not triggered | Check that `predictions-pipeline.yml` completed successfully (it sends `repository_dispatch`). Or trigger `build-blog-data.yml` manually |
| R2 upload fails | Expired Cloudflare token | Rotate `CLOUDFLARE_R2_TOKEN` in pannadata repo secrets |
| Stale predictions | RAPM/Skills pipeline not re-run | Re-run RAPM pipeline locally → Skills pipeline → `upload_prediction_caches.R` → trigger predictions pipeline |
| Missing Opta data for new league/season | Scraper not configured | Check `tier` input in `daily-opta-scrape.yml`. Tier 1 = Big 5 + UCL + WC + EURO, Tier 2 = + cups + secondary |
| `ratings-pipeline.yml` dispatch not available | Workflow not on default branch | Merge workflow file to `main` first — `workflow_dispatch` requires the file to exist on the default branch |

## 10. Cross-References

| Topic | Document |
|-------|----------|
| Pipeline flowcharts & function catalog | [`panna/CLAUDE.md`](panna/CLAUDE.md) |
| Data directory structure & league codes | [`pannadata/README.md`](pannadata/README.md) |
| Raw data column definitions | [`pannadata/DATA_DICTIONARY.md`](pannadata/DATA_DICTIONARY.md) |
| Processed data column definitions | [`panna/DATA_DICTIONARY.md`](panna/DATA_DICTIONARY.md) |
| Opta event type/qualifier IDs | [`panna/OPTA_REFERENCE.md`](panna/OPTA_REFERENCE.md) |
| Scraper overview | [`pannadata/scripts/README.md`](pannadata/scripts/README.md) |
| Opta scraper details | [`pannadata/scripts/opta/README.md`](pannadata/scripts/opta/README.md) |
| FBref scraper details | [`pannadata/scripts/fbref/README.md`](pannadata/scripts/fbref/README.md) |
| Blog data schema | [`pannadata/BLOG_DATA_SETUP.md`](pannadata/BLOG_DATA_SETUP.md) |
| Understat scraper details | [`pannadata/scripts/understat/README.md`](pannadata/scripts/understat/README.md) |
