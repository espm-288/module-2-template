# Module 2: Cloud-Native Geospatial — Reproducing the NDVI–Redlining Result

## Overview

This module has two interlocking goals:

1. **Scientific goal:** Reproduce the well-documented finding that historically redlined neighborhoods (HOLC grades C and D) have significantly lower vegetation cover (NDVI) than higher-graded areas — a signature of cumulative environmental injustice that persists decades after formal redlining ended.

2. **Methods goal:** Do this analysis using modern, scalable, cloud-native tools rather than the "download-everything" workflows of traditional GIS. The same code that works on a city should work on an entire country without modification.

Along the way, you will practice a skill that is increasingly valuable: **writing instructions that make AI coding assistants reliably produce high-quality, modern code**.

---

## Why Cloud-Native?

Traditional GIS workflows break down at scale:

| Tool | Approach | Problem at scale |
|------|----------|-----------------|
| `terra`, `stars` | Load files into RAM | Runs out of memory on large time stacks |
| `sf` on large files | Read entire file | Slow; downloads data you don't need |
| `leaflet`, `tmap` | CPU rendering | Static or limited interactivity |

Cloud-native alternatives stream data on demand and process only what you ask for:

| Tool | What it does |
|------|--------------|
| `gdalcubes` + `rstac` | Queries STAC catalogs; streams satellite data as lazy cubes; computes NDVI without touching most of the scene |
| `duckdbfs` | Reads remote vector files (FlatGeobuf, GeoParquet) lazily via DuckDB; spatial joins in-database |
| `mapgl` | WebGL maps via MapLibre GL JS; handles millions of features, 3D extrusion, smooth fly-to animations |
| `h3jsr` | H3 hexagonal indexing for fast aggregation across large point datasets |

---

## The Science: NDVI and Redlining

In the 1930s, the Home Owners' Loan Corporation (HOLC) graded US neighborhoods A–D. Grade D ("hazardous") neighborhoods were predominantly Black and immigrant communities. Banks used these maps to deny mortgages — a practice called redlining.

Studies by [Nardone et al. (2020)](https://doi.org/10.1289/EHP7936), [Hoffman et al. (2020)](https://doi.org/10.1371/journal.pone.0232224), and others show that HOLC grade D neighborhoods today have:
- Less tree canopy and lower NDVI
- Higher summer temperatures (urban heat island)
- Worse air quality
- Worse health outcomes

**Your task:** Compute mean NDVI per HOLC polygon for a city of your choice using Sentinel-2 imagery and visualize the gradient from grade A to grade D.

### Data sources

- **HOLC redlining polygons:** [Mapping Inequality](https://dsl.richmond.edu/panorama/redlining/) — download city GeoJSON, or use the [`redlining` R package](https://github.com/anthonyhaffey/redlining)
- **Sentinel-2 imagery:** AWS Earth Search STAC (`https://earth-search.aws.element84.com/v1/`) — no account needed
- **Cloud masks:** Use the Scene Classification Layer (`scl` band) to remove clouds and cloud shadows

---

## The Key Skill: Writing Agent Instructions

LLMs trained on Stack Overflow and GitHub know every R spatial package ever written. Left to their own devices, they will reach for `terra`, `leaflet`, and `tmap` because those have the most training signal. They will produce **working but outdated code**.

You can change this with a well-crafted instruction file.

### How it works

Claude Code (the CLI you are using) reads `CLAUDE.md` in the project root before every session. Think of it as a persistent system prompt you write once and the agent follows every time. Other tools use similar conventions:

| Tool | Instruction file |
|------|-----------------|
| Claude Code (CLI) | `CLAUDE.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Cursor | `.cursorrules` |
| OpenAI Codex agents | `AGENTS.md` |

The instruction file for this repo is `.github/copilot-instructions.md`. GitHub Copilot reads it automatically whenever you open Copilot Chat in this repository — no configuration needed. Open it and read it before your first coding session.

### What makes a good instruction file?

A good agent instruction file does three things:

1. **States preferences unambiguously.** "Use `mapgl`, not `leaflet`" is better than "prefer modern mapping packages."

2. **Provides concrete code templates.** The agent pattern-matches against these. A working STAC query template means the agent produces a working STAC query instead of an approximate one.

3. **Explains the *why*.** "Never use `terra::rast()` on many files — use `gdalcubes::raster_cube()` instead, which streams data without loading it into RAM" teaches the agent the principle, not just the rule, so it generalises correctly.

### Exercise: Evaluate the instructions

Before writing code, compare two Copilot Chat sessions:

1. Temporarily rename `.github/copilot-instructions.md` to something Copilot won't find (e.g., `copilot-instructions.md.bak`) and ask: *"Compute mean NDVI for each HOLC redlining polygon in Oakland using Sentinel-2 and make a choropleth map."*

2. Restore the file and ask the same question.

Compare the tool choices, code quality, and scalability of the two responses. What did the instructions change? What did they fail to change?

---

## Getting Started

### 1. Open in the devcontainer

Open this folder in VS Code and click **"Reopen in Container"** when prompted. The container builds from `rocker/geospatial:4.4` with all required packages pre-installed.

Required packages (pre-installed in the container):

```r
library(rstac)       # STAC catalog queries
library(gdalcubes)   # Cloud-native raster cubes
library(duckdbfs)    # DuckDB spatial for vector data
library(mapgl)       # MapLibre GL JS maps
library(h3jsr)       # H3 hexagonal indexing
library(sf)          # For small/local vector data
```

### 2. Open Copilot Chat

In VS Code, open the Copilot Chat panel (`Ctrl+Shift+I`). Because `.github/copilot-instructions.md` is present, Copilot will follow the tool preferences in that file for every message in this workspace.

Use **Copilot Edits** (`Ctrl+Shift+P` → "Copilot: Open Edit Session") to have Copilot write and directly edit your `.qmd` notebook as you work.

### 3. Choose your city

Pick a US city with HOLC redlining data. Good options for clear vegetation gradients:
- Oakland / San Francisco Bay Area
- Chicago
- Detroit
- Philadelphia
- Los Angeles

### 4. Work through the analysis

Suggested sequence — ask the agent to help you at each step:

1. Download or access HOLC redlining polygons for your city
2. Query Sentinel-2 STAC for summer imagery (June–August) over the city bounding box
3. Compute mean NDVI per HOLC polygon using `gdalcubes` + `extract_geom()`
4. Visualize the result with `mapgl` — try both choropleth and 3D extrusion
5. Summarize the NDVI distribution by HOLC grade (A/B/C/D) with a boxplot
6. Interpret: is the gradient what the literature predicts?

---

## Extending the Analysis

Once the core analysis works:

- **Temporal:** How has the NDVI gap changed from 2017 to 2024? (Same STAC workflow, different date range)
- **H3 aggregation:** Bin point-level tree canopy data into H3 hexagons with `h3jsr` and overlay on the redlining map
- **Multi-city:** Run the same code on 5 cities — does it scale without modification?
- **Urban heat:** Swap NDVI for Land Surface Temperature from Landsat (available on the same STAC)

---

## Getting Library Documentation into the Agent Context

LLMs may have outdated or incomplete knowledge of `gdalcubes`, `mapgl`, and `duckdbfs`. Two ways to help:

**Option 1: Paste relevant examples directly into your prompt.** Copy a working code snippet from the package vignette and say *"Here is how this function works — now apply it to my data."*

**Option 2: Use the Context7 MCP server.** [Context7](https://github.com/upstash/context7) fetches live package documentation and injects it into the agent context. The `.vscode/mcp.json` in this repo already configures it:

```json
{
  "servers": {
    "context7": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

VS Code will prompt you to start the MCP server when you open the workspace. Once running, add `use context7` to any Copilot Chat message to pull live docs, e.g.: *"use context7 to look up gdalcubes extract_geom, then apply it to compute zonal NDVI statistics."*

---

## Key Resources

- [gdalcubes tutorial (NASA TOPST)](https://boettiger-lab.github.io/nasa-topst-env-justice/tutorials/R/1-intro-R.html) — the STAC + NDVI workflow this module is based on
- [mapgl documentation](https://walker-data.com/mapgl/articles/getting-started.html)
- [duckdbfs README](https://cran.r-project.org/web/packages/duckdbfs/readme/README.html)
- [Mapping Inequality](https://dsl.richmond.edu/panorama/redlining/) — HOLC redlining data
- [Nardone et al. 2020](https://doi.org/10.1289/EHP7936) — NDVI and redlining paper
