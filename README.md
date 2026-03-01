# pannaverse

Monorepo for the panna football player rating ecosystem. Contains the R package for calculating player ratings and the data repository for cached match data.

## Repository Structure

```
pannaverse/
├── panna/        # R package - RAPM+SPM player ratings
└── pannadata/    # Data repository - cached Opta/Understat/FBref data
```

| Repository | Description | Documentation |
|------------|-------------|---------------|
| [panna](panna/) | R package implementing RAPM, SPM, and Panna ratings | [README](panna/README.md) |
| [pannadata](pannadata/) | Cached match data from Opta, Understat, and FBref | [README](pannadata/README.md) |

## Getting Started

### Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/peteowen1/pannaverse.git
```

If already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Install the Package

```r
# From local clone
devtools::install("panna")

# Or directly from GitHub
devtools::install_github("peteowen1/panna")
```

### Download Data

```r
library(panna)

# Download from GitHub releases
pb_download_source("opta")      # Opta data (primary, 15 leagues)
pb_download_source("understat") # Understat data
pb_download_source("fbref")     # FBref data
```

### Quick Example

```r
library(panna)

# Load Premier League 2024-25 Opta stats (263 columns per player)
opta_stats <- load_opta_stats("EPL", "2024-2025")

# Get aggregated player statistics
players <- player_opta_summary(
  leagues = "EPL",
  seasons = "2024-2025",
  min_minutes = 900
)
```

## What is Panna?

Panna is a player rating system that measures individual player impact on team performance:

1. **RAPM** (Regularized Adjusted Plus-Minus) - Uses lineup data from "splints" (time segments between goals/substitutions) to isolate player contributions
2. **SPM** (Statistical Plus-Minus) - Predicts RAPM from box score statistics
3. **Panna Rating** - Combines RAPM with SPM as a Bayesian prior for stability

Ratings are expressed as xG per 90 minutes above/below average, split into offensive and defensive components.

### Advanced Capabilities

- **EPV** (Expected Possession Value) - Action-level player valuation from Opta event data
- **Estimated Skills** - Bayesian decay-weighted skill estimation with position-specific priors
- **Match Predictions** - XGBoost Poisson/multinomial models for match outcome prediction

## Submodule Workflow

Both submodules track the `dev` branch. After making changes in a submodule:

```bash
# Commit in submodule (on dev branch)
cd panna
git add .
git commit -m "Your changes"
git push origin dev

# Update pannaverse to point to new commit
cd ..
git add panna
git commit -m "Update panna submodule"
```

To update submodules to latest remote:

```bash
git submodule update --remote --merge
```

## Data Sources

| Source | Leagues | Years | Key Stats |
|--------|---------|-------|-----------|
| Opta | 15 leagues | 2013+ | 263 columns, SPADL + XGBoost xG, event-level data |
| Understat | Big 5 + Russia | 2014+ | xGChain, xGBuildup |
| FBref | Big 5 + European + cups + international | 2017+ | StatsBomb xG, comprehensive passing |

## Documentation

- [panna Package Documentation](panna/README.md)
- [Data Dictionary (Pipeline)](panna/DATA_DICTIONARY.md)
- [Data Dictionary (Raw Data)](pannadata/DATA_DICTIONARY.md)

## License

MIT
