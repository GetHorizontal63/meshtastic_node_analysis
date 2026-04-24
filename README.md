# Wood County Mesh Network

An open-source planning study for a minimal resilient Meshtastic LoRa mesh network covering Wood County, West Virginia. The project combines USGS terrain data, Microsoft Building Footprints, and a custom RF propagation pipeline to site 30 relay nodes that provide emergency text messaging coverage to 86.7% of county structures — independent of cellular, internet, or grid power.

**[View the live data story →](https://your-username.github.io/wood-county-mesh)**

---

## What This Is

When commercial cellular and Wood County's public-safety trunked radio fail simultaneously — flood, ice storm, prolonged grid outage — there is no fallback messaging plane. This project designs a volunteer-deployable 915 MHz LoRa mesh using Meshtastic hardware that can carry short text and telemetry between fire stations, EOCs, hospitals, and amateur operators with no external infrastructure dependency.

The full plan converges on county-wide resilient messaging for under $5,000 in hardware (~$150/node).

---

## Repository Structure

```
/
├── index.html                          # Scrollytelling data story (main page)
├── docs/
│   └── technical_walkthrough.html     # In-depth pipeline and algorithm reference
├── data/
│   └── processed/                      # Pipeline outputs (generated locally, committed for web)
│       ├── web_bundle.json             # All map layers bundled for the frontend
│       ├── network_graph.json          # Mesh graph metadata and SPoF analysis
│       ├── building_coverage_summary.json
│       ├── hillshade.png               # Color-shaded relief overlay
│       └── hillshade.json              # Bounds metadata for Leaflet ImageOverlay
└── pipeline/
    ├── __init__.py
    ├── acquire.py                      # Download boundary, roads, buildings, POI, flood zones
    ├── terrain.py                      # Fetch USGS 3DEP 10m DEM via py3dep
    ├── analyze.py                      # DEM peak extraction, prominence scoring, building rescore
    ├── select.py                       # LoS-aware greedy set-cover node selection
    ├── rf_model.py                     # Friis link budget, Fresnel LoS, viewsheds, building coverage
    ├── redundancy.py                   # NetworkX graph analysis, SPoF detection
    ├── hillshade.py                    # Color-shaded relief PNG generation
    └── export.py                       # Bundle all processed layers into web_bundle.json
```

---

## How It Works

### Pipeline

Run the pipeline in order to regenerate all processed data from scratch:

```bash
pip install -r requirements.txt

python -m pipeline.acquire
python -m pipeline.terrain
python -m pipeline.analyze
python -m pipeline.select
python -m pipeline.rf_model
python -m pipeline.redundancy
python -m pipeline.hillshade
python -m pipeline.export
```

Each module writes to `data/processed/`. Any module can be re-run independently as long as its upstream outputs exist. If you only want to regenerate the web bundle after changing export logic:

```bash
python -m pipeline.export
```

### Key Algorithms

**Terrain Analysis (`analyze.py`)** — Extracts local maxima from the USGS 10m DEM using a 5×5 kernel. A stratified geographic grid prevents candidates from clustering on the highest ridge system. Each candidate is scored by the number of building centroids within its 14.3 km radio horizon (85%) plus local topographic prominence (15%).

**RF Model (`rf_model.py`)** — Models links using the Friis free-space path loss equation (154 dB budget at SF12/BW125) combined with 4/3-Earth curvature correction and first Fresnel zone clearance. A link is only valid if 60% of the first Fresnel radius is clear at every point along a 200-sample DEM profile. Naive endpoint-only LoS checks overestimate connectivity by 2–3×.

**Greedy Set-Cover (`select.py`)** — Frames node placement as a weighted set-cover problem over 46,406 building centroids. Each round picks the candidate that maximizes newly covered buildings, subject to: (1) clear LoS to at least one already-chosen node, and (2) 2,500 m minimum spacing from all existing nodes. The first node selected becomes the gateway anchor.

**Viewshed (`rf_model.py`)** — Shoots 1,440 radial rays per node at 0.25° resolution, walking outward in 150 m steps to find the furthest point satisfying Fresnel clearance. The union of all 30 viewsheds covers ~547 km².

**Redundancy (`redundancy.py`)** — Builds a NetworkX graph from selected nodes and clear LoS links. Identifies single points of failure (SPoF) by iterative node removal and computes all-pairs average hop count.

---

## Results

| Metric | Value |
|---|---|
| Nodes | 30 |
| Antenna height | 12 m AGL |
| Combined viewshed | ~547 km² |
| Building coverage (set-cover) | 86.7% |
| Building coverage (strict per-node LoS) | 82.7% |
| Clear inter-node links | 246 |
| Estimated hardware cost | <$5,000 |

---

## Hardware

Each node runs on commodity Meshtastic hardware:

- **Radio:** Heltec LoRa32 or RAK WisBlock
- **Antenna:** 5.8 dBi collinear, 12 m push-up mast or existing tower mount
- **Power:** Solar panel + LiFePO₄ battery, <1 W draw

Three nodes with the highest direct building visibility are designated **gateways** — connection points for volunteer-supplied internet backhaul (Starlink, residential fiber, MQTT bridge) to reach outside the county when some infrastructure survives.

---

## Data Sources

| Dataset | Source | License |
|---|---|---|
| 10m DEM | USGS 3DEP via `py3dep` | Public Domain |
| Building Footprints | Microsoft ML Buildings (WV) | ODbL |
| Roads | US Census TIGER/Line | Public Domain |
| Critical Infrastructure POI | OpenStreetMap via `osmnx` | ODbL |
| Flood Zones | FEMA NFHL ArcGIS REST API | Public Domain |

---

## Deployment Phasing

1. **Phase 1 — Gateways first.** Install the 3 gateway nodes and verify MQTT backhaul. This proves the uplink path before committing to full deployment.
2. **Phase 2 — Parkersburg-Vienna corridor.** Add relay nodes covering the highest-density population area.
3. **Phase 3 — Rural ridges.** Fill remaining county coverage with ridge-top relays.

---

## Licensing

- Code: MIT
- Data: subject to source licenses (USGS Public Domain, OSM ODbL, MS Buildings ODbL, TIGER Public Domain)

---

## Contact

This is an independent planning study, not an official Wood County or WV EMA project.

For questions, contributions, or to get involved with deployment: **mesh@example.org**
