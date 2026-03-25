# CursoKmpApp-GitGraph

A Kotlin Multiplatform project targeting Android and iOS, used as the subject of an automated dependency analysis pipeline for a computer science thesis.

---

## Project Structure

- `/composeApp` — Shared code across Compose Multiplatform applications.
  - `commonMain` — Code common to all targets.
  - `androidMain` — Android-specific code.
  - `iosMain` — iOS-specific code.
- `/iosApp` — iOS application entry point. Add SwiftUI code here.
- `/pipeline` — Automated thesis pipeline scripts (SBOM retrieval, analysis, visualization).

Learn more about [Kotlin Multiplatform](https://www.jetbrains.com/help/kotlin-multiplatform-dev/get-started.html).

---

## Thesis Pipeline

This repository includes an automated pipeline that retrieves and analyzes its own dependency graph as part of a computer science thesis. The pipeline is located in the [`pipeline/`](pipeline/) directory and is fully independent from the KMP application source code.

---

## Step 1 — SBOM Retrieval

### What is an SBOM?

A **Software Bill of Materials (SBOM)** is a machine-readable inventory of every dependency in a software project. GitHub generates SBOMs in **SPDX 2.3** format via its Dependency Graph feature.

This repository has **610 tracked dependencies** visible under Insights → Dependency graph.

---

### GitHub API Endpoint

```
GET https://api.github.com/repos/{owner}/{repo}/dependency-graph/sbom
```

For this repository:

```
GET https://api.github.com/repos/Sreys54/CursoKmpApp-GitGraph/dependency-graph/sbom
```

The response is a JSON object with a top-level `sbom` key containing a full SPDX 2.3 document.

---

### Authentication

A Personal Access Token (PAT) is required. Two token types are supported:

| Token Type | Required Permission | Notes |
|---|---|---|
| Classic PAT | `repo` scope (private repos) or no scope (public) | Easier to set up |
| Fine-grained PAT | `dependency_graph: read` | More secure, recommended |

**Required request headers:**

```
Authorization: Bearer <YOUR_TOKEN>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
```

---

### Rate Limits

| Auth State | Limit |
|---|---|
| Unauthenticated | 60 requests/hour per IP |
| Authenticated (PAT) | 5,000 requests/hour per user |

The scripts log `X-RateLimit-Remaining` and `X-RateLimit-Reset` on every run. The SBOM is saved locally to avoid redundant API calls.

---

### Prerequisites

1. Enable the Dependency Graph: go to your repository → **Settings → Security → Dependency graph**.
2. Create a PAT at [github.com/settings/tokens](https://github.com/settings/tokens):
   - Classic: check the `repo` scope.
   - Fine-grained: grant `dependency_graph: read` for this repository.
3. Set the three required environment variables (see below).

---

### Files

```
pipeline/sbom/
├── fetch_sbom.py      — Python script (recommended)
├── fetch_sbom.sh      — curl/bash script
├── fetch_sbom.js      — Node.js script (zero npm dependencies)
├── .env.example       — Template for credentials
└── requirements.txt   — Python dependency (requests)
```

> `sbom.json` and `.env` are excluded from git via `.gitignore`.

---

### Environment Variables

Copy `.env.example` to `.env` and fill in your values. **Never commit the `.env` file.**

| Variable | Required | Description |
|---|---|---|
| `GITHUB_TOKEN` | Yes | Personal Access Token |
| `GITHUB_OWNER` | Yes | Repository owner (`Sreys54`) |
| `GITHUB_REPO` | Yes | Repository name (`CursoKmpApp-GitGraph`) |
| `SBOM_OUTPUT_PATH` | No | Output file path (default: `sbom.json`) |

```bash
export GITHUB_TOKEN="ghp_your_token_here"
export GITHUB_OWNER="Sreys54"
export GITHUB_REPO="CursoKmpApp-GitGraph"
```

---

### Option A — Python (recommended)

**Install dependency:**

```bash
pip install -r pipeline/sbom/requirements.txt
```

**Run:**

```bash
cd pipeline/sbom
python fetch_sbom.py
```

**What each part does:**

| Function | Purpose |
|---|---|
| `load_config()` | Reads env vars; exits immediately with a clear message if any are missing |
| `build_headers()` | Constructs the three required GitHub API headers |
| `fetch_sbom()` | Makes the HTTP call, checks status code, validates response shape |
| `save_sbom()` | Writes pretty-printed JSON to `sbom.json` |
| `print_summary()` | Prints package count and a sample list to stdout |
| Rate limit logging | Logs `X-RateLimit-Remaining` every run |

---

### Option B — curl / bash

```bash
cd pipeline/sbom
bash fetch_sbom.sh
```

**What each part does:**

| Section | Purpose |
|---|---|
| `set -euo pipefail` | Exits immediately on any error, unset variable, or pipe failure |
| Variable validation loop | Checks all required env vars before making any network call |
| `curl --write-out "%{http_code}"` | Captures HTTP status separately from the response body |
| Status code checks | Specific messages for 403 (token/permissions) and 404 (wrong repo) |
| Inline Python summary | Uses the system `python3` to print a human-readable summary |

---

### Option C — Node.js

No npm install needed — uses only built-in Node.js modules (`https`, `fs`).

```bash
cd pipeline/sbom
node fetch_sbom.js
```

**What each part does:**

| Function | Purpose |
|---|---|
| `loadConfig()` | Reads env vars; exits immediately with a clear message if any are missing |
| `httpGet()` | Native HTTPS GET with a 30-second timeout; no external dependencies |
| `fetchSbom()` | Calls the API, logs rate-limit headers, validates response shape |
| `saveSbom()` | Writes pretty-printed JSON to disk |
| `printSummary()` | Prints package count and a sample list to stdout |

---

### Example Output

```
[INFO] Fetching SBOM from: https://api.github.com/repos/Sreys54/CursoKmpApp-GitGraph/dependency-graph/sbom
[INFO] Rate limit — remaining: 4998, resets at unix ts: 1742860800
[INFO] SBOM saved to 'sbom.json'

========== SBOM Summary ==========
  SPDX Version   : SPDX-2.3
  Document name  : Sreys54/CursoKmpApp-GitGraph
  Created        : 2026-03-24T00:00:00Z
  Total packages : 610

  Sample (first 8 packages):
    - androidx.activity:activity @ 1.7.0
    - androidx.activity:activity @ 1.8.2
    - androidx.activity:activity-compose @ 1.8.2
    - androidx.activity:activity-ktx @ 1.7.0
    - androidx.annotation:annotation @ 1.7.1
    - androidx.arch.core:core-common @ 2.2.0
    - androidx.arch.core:core-runtime @ 2.2.0
    - androidx.collection:collection @ 1.4.0
===================================

[INFO] Done.
```

---

### Error Handling

| HTTP Code | Cause | Message |
|---|---|---|
| 403 | Wrong token scope or Dependency Graph disabled | Check token scopes and repo settings |
| 404 | Wrong owner/repo name | Verify `GITHUB_OWNER` and `GITHUB_REPO` |
| Other | Network or API error | Raw response body is printed |

---

### Design Decisions

| Decision | Reason |
|---|---|
| Separate `pipeline/` directory | Keeps thesis tooling isolated from the KMP app source |
| Environment variables only — no hardcoded values | Safe to commit; no credentials ever in git history |
| Specific 403/404 error messages | These are the two most common failure modes |
| Rate limit logged on every run | Visible in CI logs before throttling becomes a problem |
| `sbom.json` excluded from git | Output is reproducible on demand; no need to version it |
| Node.js uses only built-in modules | Zero-dependency script; works anywhere Node.js is installed |

---

## Step 2 — SBOM to Graph Transformation

### SPDX 2.3 Structure

GitHub's SBOM uses the **SPDX 2.3** standard (not CycloneDX). The JSON file has two key arrays:

#### `packages` — the nodes

Every dependency detected in the repository. Each entry contains:

| Field | Description | Example |
|---|---|---|
| `SPDXID` | Unique internal ID used to link relationships | `"SPDXRef-maven-org.jetbrains.kotlin-kotlin-stdlib-1.9.23-763561"` |
| `name` | Human-readable package name | `"org.jetbrains.kotlin:kotlin-stdlib"` |
| `versionInfo` | Resolved version string | `"1.9.23"` |
| `licenseConcluded` | SPDX license expression (may be absent) | `"Apache-2.0"` |
| `externalRefs` | Array of references; the `purl` entry gives ecosystem info | `"pkg:maven/org.jetbrains.kotlin/kotlin-stdlib@1.9.23"` |

This repository's SBOM contains **611 packages**.

#### `relationships` — the edges

Every directed dependency edge between packages. Each entry contains:

| Field | Description |
|---|---|
| `spdxElementId` | The package **that has** the dependency (source) |
| `relatedSpdxElement` | The package **being depended on** (target) |
| `relationshipType` | `"DEPENDS_ON"` for all dependency edges; one `"DESCRIBES"` entry links the document root to the repo (ignored) |

This repository's SBOM contains **3,038 relationships** — 3,037 `DEPENDS_ON` edges and 1 `DESCRIBES` entry.

A directed edge `A → B` means **"A depends on B"**.

---

### Transformation Pipeline

The script `transform_sbom.js` executes five sequential steps:

```
sbom.json
    │
    ▼
[1] Load & validate SPDX JSON
    │
    ▼
[2] Build SPDXID → "name@version" index   (611 entries)
    │
    ▼
[3] Extract DEPENDS_ON edges              (3,037 links)
    │   skip: DESCRIBES, unresolved, self-loops, duplicates
    ▼
[4] Collect connected nodes               (611 nodes)
    │
    ▼
[5] Write graph.json
```

---

### Output Format

```json
{
  "nodes": [
    { "id": "org.jetbrains.kotlin:kotlin-stdlib-common@1.9.23" },
    { "id": "org.jetbrains.kotlin:kotlin-stdlib@1.9.23" }
  ],
  "links": [
    {
      "source": "org.jetbrains.kotlin:kotlin-stdlib-common@1.9.23",
      "target": "org.jetbrains.kotlin:kotlin-stdlib@1.9.23"
    }
  ]
}
```

Node IDs use the format `name@version` (e.g., `org.jetbrains.kotlin:kotlin-stdlib@1.9.23`). This is human-readable, unique per package version, and directly usable as a D3 node identifier.

---

### Files

```
pipeline/sbom/
├── fetch_sbom.py        — Step 1: retrieve sbom.json from GitHub API
├── fetch_sbom.sh        — Step 1: curl/bash alternative
├── fetch_sbom.js        — Step 1: Node.js alternative
├── transform_sbom.js    — Step 2: transform sbom.json → graph.json
├── .env.example         — Credentials template
└── requirements.txt     — Python dependency (requests)
```

> `graph.json` is excluded from git — it is always derived from `sbom.json`.

---

### Run

```bash
cd pipeline/sbom

# Using default paths (sbom.json → graph.json)
node transform_sbom.js

# Using custom paths
SBOM_INPUT_PATH=/path/to/sbom.json GRAPH_OUTPUT_PATH=graph.json node transform_sbom.js
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `SBOM_INPUT_PATH` | No | Input SBOM file (default: `sbom.json`) |
| `GRAPH_OUTPUT_PATH` | No | Output graph file (default: `graph.json`) |

The script auto-detects whether the SBOM is the raw GitHub export or the API-wrapped version (with a top-level `sbom` key) and handles both.

---

### What each function does

| Function | Purpose |
|---|---|
| `loadSbom()` | Reads and parses the JSON; unwraps the API `sbom` wrapper if present; validates required keys |
| `buildPackageIndex()` | Creates a `Map` from `SPDXID → "name@version"` so relationships can be resolved to readable labels |
| `extractEdges()` | Filters `DEPENDS_ON` relationships; resolves both endpoints via the index; drops self-loops and duplicates |
| `buildNodes()` | Collects every node ID that appears in at least one edge (connected nodes only) |
| `saveGraph()` | Assembles `{ nodes, links }` and writes to disk as pretty-printed JSON |
| `printSummary()` | Computes in-degree for each node; reports top 5 most depended-upon packages |

---

### Edge Cases Handled

| Case | Behavior |
|---|---|
| `DESCRIBES` relationship | Skipped — it is a document metadata entry, not a package dependency |
| SPDXID in a relationship not found in packages | Skipped with a `[WARN]` log |
| Package with no `name` field | Falls back to the `SPDXID` string |
| Package with no `versionInfo` | Node ID is just `name` (no `@version` suffix) |
| Self-loops (`source === target`) | Skipped silently |
| Duplicate edges | Deduplicated via a `Set` |
| API-wrapped SBOM (top-level `sbom` key) | Auto-detected and unwrapped |

---

### Example Output

```
[INFO] Reading SBOM from: sbom.json
[INFO] SBOM loaded — 611 packages, 3038 relationships.
[INFO] Package index built — 611 entries.
[INFO] Edges extracted — 3037 unique dependency links.
[INFO] Nodes collected — 611 connected packages.
[INFO] Graph saved to: graph.json

========== Graph Summary ==========
  Total nodes (packages) : 611
  Total links (edges)    : 3037

  Top 5 most depended-upon packages:
    [103 dependents] org.jetbrains.kotlin:kotlin-stdlib@1.9.22
    [97 dependents]  org.jetbrains.kotlin:kotlin-stdlib-common@1.9.23
    [89 dependents]  org.jetbrains.kotlinx:kotlinx-coroutines-core@1.8.0
    [86 dependents]  org.jetbrains.kotlinx:atomicfu@0.23.2
    [81 dependents]  org.jetbrains.kotlin:kotlin-stdlib@1.9.23
====================================

[INFO] Done. graph.json is ready for D3 visualization.
```

---

### Design Decisions

| Decision | Reason |
|---|---|
| Node ID = `name@version` | Human-readable, unique per version, matches purl convention |
| Connected nodes only (no isolated nodes) | Isolated nodes add visual clutter to arc diagrams without contributing structure |
| `SPDXID` as the internal key | Relationships reference packages by `SPDXID`, not by name — the index is the bridge |
| Auto-detect API wrapper | The GitHub API adds a top-level `sbom` key; the raw export does not — both are supported |
| Deduplication via `Set` | Prevents duplicate edges that would distort D3 force calculations |
| In-degree summary | Identifies the most critical shared dependencies at a glance |

---

## Step 3 — Arc Diagram Visualization

### ObservableHQ vs. local D3

[ObservableHQ](https://observablehq.com/@d3/arc-diagram) is an interactive notebook platform for exploring D3 visualizations. It does **not** provide an automation API — you cannot call it programmatically, pipe data into it, or generate output files from a script. It is a manual, browser-only environment.

**For an automated pipeline the correct approach is:**

> Serve a static HTML file that loads `graph.json` at runtime using the browser's `fetch()` API.

No build tools or npm packages are required. D3 v7 is loaded from a CDN. A one-line HTTP server command is all that is needed to run it.

---

### How an Arc Diagram Works

An arc diagram maps a graph onto a single axis:

```
  ╭──────────────────────╮       ← arc (quadratic Bézier curve)
  │            ╭─────╮   │
──●────●────●──●─────●───●──●── ← baseline (the x-axis)
  A    B    C  D     E   F  G    ← nodes (packages)
```

| Concept | Meaning in this diagram |
|---|---|
| **Baseline** | A horizontal line where all nodes are placed |
| **Node position** | Determined by sort order (degree, ecosystem, or name) |
| **Arc** | A Bézier curve above the baseline connecting two dependent packages |
| **Arc height** | Proportional to horizontal distance — long-range deps create tall arcs |
| **Arc color** | Matches the source node's ecosystem |
| **Node size** | Larger circle = more connections (degree) |

A directed edge `A → B` ("A depends on B") appears as an arc from A to B. Because many packages share common dependencies (like `kotlin-stdlib`), those packages accumulate many arcs, making them visually dominant.

---

### File

```
pipeline/visualization/
└── arc_diagram.html   — self-contained D3 arc diagram
```

The HTML file loads `../sbom/graph.json` via `fetch()`. Both files must be served from the same origin (a local HTTP server).

---

### How to Run Locally

**Option A — Node.js `serve` (recommended):**

```bash
# Install once globally
npm install -g serve

# Serve the pipeline directory and open the visualization
serve pipeline/
# Then open: http://localhost:3000/visualization/arc_diagram.html
```

**Option B — Python HTTP server:**

```bash
# From the project root
python -m http.server 8080 --directory pipeline/
# Then open: http://localhost:8080/visualization/arc_diagram.html
```

**Option C — VS Code Live Server extension:**

Right-click `arc_diagram.html` → "Open with Live Server".
Make sure the server root is set to the `pipeline/` directory so the relative path `../sbom/graph.json` resolves correctly.

> **Why can't I just open the HTML file directly?**
> Browsers block `fetch()` requests to local files (`file://` protocol) due to the same-origin security policy. A local HTTP server bypasses this restriction.

---

### Controls

| Control | Description |
|---|---|
| **Min. degree slider** | Hides packages with fewer connections than the threshold. Default: 10 (shows 179 nodes). Raise it to focus on the most connected hubs. |
| **Sort by: Degree** | Places the highest-degree nodes in the center. Long-range arcs span across the diagram — hub packages become visually obvious. |
| **Sort by: Ecosystem** | Groups packages by organization prefix (`org.jetbrains`, `io.ktor`, etc.). Arcs within a group stay short; cross-group dependencies create tall arcs. |
| **Sort by: Name (A–Z)** | Alphabetical order. Useful for finding a specific package. |

---

### Interaction

| Action | Effect |
|---|---|
| Hover over a node | Highlights all arcs connected to that node; dims everything else |
| Tooltip | Shows package name, version, ecosystem, and total degree |
| Move mouse | Tooltip follows the cursor |
| Mouse leave | Resets all highlighting |

---

### Code Structure

| Section | What it does |
|---|---|
| `getEcosystem(id)` | Extracts the group prefix from `"group:artifact@version"` for coloring |
| `colorOf(id)` | Maps ecosystem → fixed hex color using `ECOSYSTEM_COLORS` |
| `fetch("../sbom/graph.json")` | Loads the graph data; shows a clear error if the file is missing or the server isn't running |
| `boot(data)` | Runs once after load: computes degree, builds legend, wires controls |
| `render(data, degree, minDegree, sortBy)` | Called every time a control changes: filters, sorts, and re-draws the entire diagram |
| `arcPath(sourceId, targetId)` | Returns the SVG `d` attribute for a quadratic Bézier arc above the baseline |
| Hover handlers | Build an adjacency `Set` per node; use D3 `.classed()` to toggle `dimmed` / `highlighted` CSS classes |

---

### Data at a Glance (this repository)

| Metric | Value |
|---|---|
| Total packages (nodes) | 611 |
| Total dependency edges | 3,037 |
| Most connected package | `com.github.Sreys54/CursoKmpApp-GitGraph@main` (249 connections) |
| Most depended-upon | `kotlin-stdlib@1.9.22` (103 dependents) |
| Dominant ecosystem | `org.jetbrains` (205 packages — 34% of all nodes) |
| Default view (degree ≥ 10) | 179 nodes visible |

---

### Design Decisions

| Decision | Reason |
|---|---|
| Static HTML + CDN D3 | No build step, no npm, works anywhere — fits an automated pipeline |
| `fetch()` loads graph.json at runtime | The diagram always reflects the latest SBOM without re-embedding data |
| Quadratic Bézier arcs (not SVG arc command) | Simpler math, height naturally proportional to node distance |
| Arc height = distance × 0.45 | Keeps arcs within a visually comfortable height; avoids overlapping the top of the SVG |
| Interleaved degree sort (hub in center) | The most connected node in the center distributes its arcs symmetrically — cleaner than placing it at one edge |
| Dimming on hover (not hiding) | Dimmed nodes give context; hidden nodes make the diagram feel broken |
| Min-degree filter | 611 nodes with all 3,037 edges is unreadable — the filter exposes meaningful structure at any zoom level |
| Color = source ecosystem | Immediately shows whether a dependency crosses ecosystem boundaries |
