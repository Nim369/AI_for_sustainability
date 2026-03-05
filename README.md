# AI for Sustainability — SRIP 2026
## Earth Observation Pipeline: Delhi Airshed Land-Use Classification

> **Disclosure:** AI tools (Claude by Anthropic) were used to assist in writing this code.
> I fully understand the code and can explain every section during the one-on-one discussion.

---

## Project Structure

```
project/
├── data/
│   ├── delhi_ncr.shp            # Delhi-NCR region shapefile
│   ├── delhi_airshed.shp        # Delhi Airshed shapefile
│   ├── land_cover.tif           # ESA WorldCover 2021 (10m)
│   ├── sentinel2_patches/       # .png image patches (128×128)
│   └── image_coords.csv         # filename, latitude, longitude
│
├── outputs/                     # Auto-created at runtime
│   ├── q1_grid_overlay.png
│   ├── q2_class_distribution.png
│   ├── q3_training_curves.png
│   ├── q3_confusion_matrix.png
│   ├── labels.csv
│   └── resnet18_landuse.pth
│
├── srip_solution.py             # Main pipeline
└── README.md
```

---

## Setup

```bash
pip install geopandas rasterio shapely matplotlib numpy pandas \
            scikit-learn torch torchvision pillow scipy
```

---

## Running

```bash
python srip_solution.py
```

Make sure to update the **FILE PATHS** section at the top of `srip_solution.py` to point to your actual data files.

---

## Pipeline Overview

### Q1 — Spatial Reasoning & Data Filtering
- Loads Delhi-NCR shapefile and reprojects to EPSG:4326
- Builds a **60×60 km uniform grid** in EPSG:32644 (UTM Zone 44N)
- Overlays the grid on the NCR region using matplotlib
- Filters satellite image coordinates falling **inside** the NCR boundary
- Reports counts before and after filtering

### Q2 — Label Construction & Dataset Preparation
- For each filtered image, extracts a **128×128 land-cover patch** from `land_cover.tif` centred on the image's coordinates
- Assigns the label as the **dominant (mode) ESA class** in the patch
- Maps ESA class codes → simplified categories: **Built-up, Vegetation, Cropland, Water, Others**
- Performs a **60/40 stratified train-test split**
- Visualises class distribution for both splits

### Q3 — Model Training & Supervised Evaluation
- Trains a **ResNet-18** (pretrained, fine-tuned) on the labelled Sentinel-2 patches
- Uses data augmentation: random flips, colour jitter
- Trains for 15 epochs with Adam optimizer + StepLR scheduler
- Reports **Accuracy** and **weighted F1-score** on the test set
- Displays a **confusion matrix** with interpretation

---

## ESA WorldCover 2021 Class Mapping

| ESA Code | Original Class      | Simplified Label |
|----------|---------------------|-----------------|
| 10       | Tree cover          | Vegetation       |
| 20       | Shrubland           | Vegetation       |
| 30       | Grassland           | Vegetation       |
| 40       | Cropland            | Cropland         |
| 50       | Built-up            | Built-up         |
| 60       | Bare/sparse veg.    | Others           |
| 70       | Snow and ice        | Others           |
| 80       | Water bodies        | Water            |
| 90       | Herbaceous wetland  | Others           |
| 95       | Mangroves           | Others           |
| 100      | Moss and lichen     | Others           |

---

## Key Design Decisions

1. **Grid CRS:** EPSG:32644 (UTM Zone 44N) is used for building the 60 km grid because it preserves metric distances. The grid is then reprojected to EPSG:4326 for plotting.

2. **Label extraction:** `rasterio.windows.Window` with `boundless=True` safely handles images near the raster boundary by zero-padding.

3. **Mode computation:** Uses `scipy.stats.mode` on non-zero pixels only to avoid nodata contamination.

4. **Model:** ResNet-18 with ImageNet pretrained weights. Only the final FC layer is replaced, keeping feature extraction efficient for a small dataset. A custom CNN option is also provided.

5. **Stratified split:** Ensures each class is proportionally represented in train and test sets.
