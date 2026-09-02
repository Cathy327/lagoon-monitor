# Technical Documentation — Lagoon Water Quality Monitor

## System Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Sentinel-2  │────▶│  Google      │────▶│  Google      │
│  (ESA)       │     │  Earth       │     │  Drive       │
│              │     │  Engine      │     │  (persistent │
└──────────────┘     └──────────────┘     │   storage)   │
                           │              └──────┬───────┘
                     ┌─────┴─────┐               │
                     │  Google   │───────────────┘
                     │  Colab    │
                     │  Notebook │──────▶ ZIP download
                     └───────────┘              │
                                          ┌─────┴─────┐
                                          │  Local     │
                                          │  Dashboard │
                                          │  (HTML)    │
                                          └───────────┘
```

The pipeline has three components:

1. **Google Earth Engine**: Processes Sentinel-2 imagery, computes water quality indices, renders visualisations.
2. **Google Colab + Drive**: Orchestrates the export pipeline, stores data persistently on Drive, packages results for download.
3. **Local Dashboard**: Static HTML file that reads exported data and renders an interactive visualisation.

---

## Data Source

- **Collection**: `COPERNICUS/S2_SR_HARMONIZED` (Sentinel-2 Level-2A Surface Reflectance)
- **Spatial resolution**: 10 m (Bands B4, B5, B8), 20 m (Band B11)
- **Revisit time**: ~5 days (combined S2A + S2B)
- **Cloud filter**: Scenes with `CLOUDY_PIXEL_PERCENTAGE` > 30% are excluded
- **Study area**: Lagoon polygon digitised in QGIS, uploaded as GEE asset at `projects/lagoon-dashboard-507406/assets/lagoon`
- **Temporal range**: 2024-01-01 to present

---

## Index Calculations

### FAI (Floating Algae Index)

Detects floating vegetation and algal scum on the water surface using a baseline subtraction approach.

```
Red  = B4 × 0.0001    (665 nm)
NIR  = B8 × 0.0001    (842 nm)
SWIR = B11 × 0.0001   (1610 nm)

FAI = NIR − [Red + (SWIR − Red) × (842 − 665) / (1610 − 665)]
```

**Visualisation**: min = −0.05, max = 0.15, palette = blue → cyan → green → yellow → orange → red

### NDCI (Normalised Difference Chlorophyll Index)

Estimates chlorophyll-a concentration using the red-edge spectral region.

```
NDCI = (B5 − B4) / (B5 + B4)
```

Implemented via `ee.Image.normalizedDifference(['B5', 'B4'])`. No manual reflectance scaling required as the ratio cancels the scale factor.

**Visualisation**: min = −0.1, max = 0.5, palette = blue → cyan → green → yellow → orange → red

---

## Same-Day Image Compositing

Sentinel-2 operates two satellites (S2A and S2B) that may both image the study area on the same day from different orbits. When multiple images exist for the same date:

- **Rendered images**: Pixel-wise mean composite (`.mean()`) before visualisation and export. One output file per date.
- **Time series statistics**: Composite before computing zonal statistics. One CSV row per date.

---

## Export Pipeline (Colab Notebook)

### Workflow

1. **Mount Google Drive** and authenticate with GEE.
2. **Detect existing data** on Drive by scanning the image folder.
3. **Query GEE** for new imagery since the latest existing date.
4. **Download new images** via `ee.Image.getDownloadURL()`:
   - Compute FAI/NDCI for each date.
   - Apply colour palette via `.visualize()`.
   - Add alpha channel (transparent outside lagoon boundary).
   - Download as GeoTIFF, convert to PNG with transparency using rasterio and Pillow.
   - Save directly to Google Drive.
5. **Regenerate full CSV** by computing zonal statistics (mean + standard deviation) for all dates.
6. **Generate metadata**: `dates.json`, `bounds.json`, `dashboard_data.js`.
7. **Package and download** as a zip file.

### Incremental Update Logic

The notebook automatically detects which dates already have exported images on Google Drive and only processes new dates. The CSV and metadata files are regenerated from scratch each time to ensure completeness.

### Rate Limiting and Error Handling

- 2-second delay between GEE API calls to avoid rate limits.
- Automatic retry up to 3 times on download failure with 30-second intervals.
- Failed exports are logged; the notebook continues with remaining dates.

---

## Output Files

### Directory Structure

```
data/
├── images/
│   ├── fai/
│   │   ├── fai_2024-01-10.png
│   │   ├── fai_2024-01-30.png
│   │   └── ...
│   └── ndci/
│       ├── ndci_2024-01-10.png
│       ├── ndci_2024-01-30.png
│       └── ...
├── full_timeseries.csv
├── bounds.json
├── dates.json
└── dashboard_data.js
```

### File Formats

**Rendered images** (PNG, RGBA):
- Clipped to lagoon polygon boundary.
- Pixels outside the lagoon are transparent (alpha = 0).
- Colour palette applied by GEE `.visualize()`.
- Naming convention: `{index}_{YYYY-MM-DD}.png`.
- Typical file size: 20–50 KB each.

**full_timeseries.csv**:
| Column | Type | Description |
|--------|------|-------------|
| date | string (YYYY-MM-DD) | Acquisition date |
| FAI_mean | float | Spatial mean of FAI across the lagoon |
| FAI_stdDev | float | Spatial standard deviation of FAI |
| NDCI_mean | float | Spatial mean of NDCI across the lagoon |
| NDCI_stdDev | float | Spatial standard deviation of NDCI |

**bounds.json**:
```json
{
  "southwest": [143.84538, -37.61234],
  "northeast": [143.85009, -37.60359]
}
```

**dates.json**: Sorted array of all available date strings.

**dashboard_data.js**: JavaScript file containing `DATES`, `BOUNDS`, and `TIMESERIES` as global constants, enabling the dashboard to load data without fetch requests (required for local file:// operation).

---

## Dashboard Frontend

### Technology

- **Leaflet.js** (v1.9.4): Map rendering with CartoDB Light basemap tiles.
- **Leaflet Gesture Handling** (v1.2.2): Prevents map from intercepting page scroll.
- **Canvas API**: Time series charts with error bands, drawn directly on HTML5 canvas.
- **localStorage**: Bloom event records persist in the browser.

### Modules

1. **Satellite Imagery Browser**: Side-by-side FAI/NDCI maps with date dropdown and prev/next navigation. Images loaded as `L.imageOverlay` positioned using bounding box coordinates.

2. **Three-Year Comparison**: User selects an end date; the dashboard filters available imagery into a one-month window for each year (2024, 2025, 2026). Thumbnails are clickable for enlarged view.

3. **Trend Charts**: Full time series from 2024 to present. Mean line with standard deviation band. Comparison window highlighted with coloured background regions. Bloom event markers drawn at their chronological position on the time axis.

4. **Bloom Event Log**: Manual recording of observed bloom events with date, severity, and notes. Stored in `localStorage`. Markers rendered on trend charts.

### Offline Capability

The dashboard works offline except for map background tiles (which require internet). All satellite imagery overlays, charts, and data are loaded from local files.

---

## Google Account and GEE Project

- **Google Account**: A dedicated account was created for this project.
- **GEE Cloud Project**: `lagoon-dashboard-507406`
- **GEE Asset**: `projects/lagoon-dashboard-507406/assets/lagoon` (lagoon boundary polygon)

The Colab notebook uses this project for GEE authentication and asset access.

---

## Observed Data Characteristics

- **Total valid images**: ~75 dates (as of August 2026)
- **FAI mean range**: −0.003 to 0.105
- **NDCI mean range**: 0.012 to 0.450
- **Seasonal gaps**: Winter months (June–August) often have extended gaps due to cloud cover. The period August–October 2024 had a ~3 month gap.
- **Latest data latency**: Sentinel-2 processing delay is typically 2–5 days. Extended gaps are caused by cloud cover, not processing delay.
