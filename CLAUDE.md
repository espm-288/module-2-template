# Agent Instructions: Cloud-Native Geospatial in R

You are assisting with cloud-native geospatial analysis in R focused on environmental justice research. These instructions define the exact tools and patterns to use. Do not substitute alternative packages.

## Tool Preferences

### Raster data → `gdalcubes` + `rstac`

**Always** use `gdalcubes` for raster operations. Never use `terra`, `stars`, or `raster` for anything that can be done with `gdalcubes`. `gdalcubes` streams data via STAC without loading it into RAM and scales to terabytes; `terra`/`stars` load everything into memory and fail on large datasets.

**Standard STAC → NDVI workflow:**

```r
library(rstac)
library(gdalcubes)

# 1. Query STAC catalog
items <- stac("https://earth-search.aws.element84.com/v1/") |>
  stac_search(
    collections = "sentinel-2-c1-l2a",
    bbox        = c(lon_min, lat_min, lon_max, lat_max),
    datetime    = "2023-06-01/2023-08-31",
    limit       = 200
  ) |>
  ext_query("eo:cloud_cover" < 20) |>
  post_request() |>
  items_fetch()

# 2. Build image collection (select only the bands you need)
col <- stac_image_collection(
  items$features,
  asset_names = c("red", "nir", "scl")   # Sentinel-2 C1 uses lowercase band names
)

# 3. Define cube view — resolution, CRS, time
v <- cube_view(
  srs         = "EPSG:4326",
  extent      = list(
    t0 = "2023-06-01", t1 = "2023-08-31",
    left = lon_min, right = lon_max,
    bottom = lat_min, top = lat_max
  ),
  dx = 0.0002, dy = 0.0002,   # ~20 m at mid-latitudes
  dt = "P1M",                  # monthly composites
  aggregation = "median",
  resampling  = "bilinear"
)

# 4. Apply cloud mask and compute NDVI
S2.mask <- image_mask("scl", values = c(3, 8, 9))  # cloud shadow, cloud, cirrus

ndvi <- raster_cube(col, v, mask = S2.mask) |>
  select_bands(c("red", "nir")) |>
  apply_pixel("(nir - red) / (nir + red)", "NDVI") |>
  reduce_time(c("mean(NDVI)"))           # collapse to single time slice

# 5. Zonal statistics with sf polygons
holc_sf <- sf::read_sf("holc_redlining.geojson")
zonal   <- ndvi |> extract_geom(holc_sf, FUN = mean, reduce_time = TRUE)
holc_sf$mean_NDVI <- zonal$NDVI
```

---

### Vector data → `duckdbfs` (large/remote) or `sf` (small/local)

Use `duckdbfs` when data is remote, large, or you need fast spatial joins. `duckdbfs` is lazy by default — nothing downloads until `collect()` or `to_sf()`.

```r
library(duckdbfs)

# Read remote spatial file lazily (FlatGeobuf, GeoParquet, GeoJSON, etc.)
redlining <- open_dataset(
  "https://example.com/holc_redlining.fgb",
  format = "sf"
)

# Filter before collecting — only pulls matching rows
oakland <- redlining |>
  filter(city == "Oakland") |>
  to_sf(crs = 4326)

# Spatial join: which points fall within which polygons?
result <- spatial_join(points_tbl, polygons_tbl, by = "st_within")

# Create geometry from lat/lon columns
pts <- open_dataset("data.csv", format = "csv") |>
  mutate(geometry = st_point(longitude, latitude)) |>
  to_sf(crs = 4326)
```

---

### Maps → `mapgl` (always)

**Never** use `tmap`, `leaflet`, `mapview`, or `ggplot2` for maps. Use `mapgl`, which renders via MapLibre GL JS (WebGL). `maplibre()` requires no API key.

**Choropleth (e.g. HOLC grades):**

```r
library(mapgl)

maplibre(
  style  = carto_style("positron"),
  center = c(-122.25, 37.82),
  zoom   = 12
) |>
  add_fill_layer(
    id      = "redlining",
    source  = holc_sf,
    fill_color = match_expr(
      column = "holc_grade",
      values = list("A",       "B",       "C",       "D"),
      stops  = list("#76a865", "#7cb5bd", "#ffff00", "#d9838d")
    ),
    fill_opacity      = 0.65,
    fill_outline_color = "white"
  ) |>
  add_tooltips("redlining", "holc_grade") |>
  add_legend(
    "HOLC Grade",
    values = c("A", "B", "C", "D"),
    colors = c("#76a865", "#7cb5bd", "#ffff00", "#d9838d"),
    type   = "categorical"
  )
```

**Continuous fill (e.g. mean NDVI per polygon):**

```r
maplibre(style = carto_style("positron"), center = c(-122.25, 37.82), zoom = 12) |>
  add_fill_layer(
    id     = "ndvi",
    source = holc_sf,
    fill_color = interpolate(
      column = "mean_NDVI",
      values = c(0, 0.3, 0.6),
      stops  = c("#d73027", "#ffffbf", "#1a9850")
    ),
    fill_opacity = 0.8
  ) |>
  add_legend("Mean NDVI", values = c(0, 0.6), colors = c("#d73027", "#1a9850"))
```

**3D extrusion (height encodes a variable):**

```r
maplibre(
  style = carto_style("dark-matter"),
  center = c(-122.25, 37.82),
  zoom = 12,
  pitch = 50
) |>
  add_fill_extrusion_layer(
    id     = "ndvi-3d",
    source = holc_sf,
    fill_extrusion_color = interpolate(
      column = "mean_NDVI",
      values = c(0, 0.6),
      stops  = c("#d73027", "#1a9850")
    ),
    fill_extrusion_height = interpolate(
      column = "mean_NDVI",
      values = c(0, 0.6),
      stops  = c(0, 800)
    ),
    fill_extrusion_opacity = 0.85
  )
```

---

### H3 spatial indexing → `h3jsr`

Use `h3jsr` for hexagonal binning and H3-indexed joins.

```r
library(h3jsr)

# Convert sf points to H3 cell IDs at resolution 9 (~174 m)
pts$h3 <- point_to_cell(pts, res = 9)

# Aggregate by H3 cell
hex_counts <- pts |>
  group_by(h3) |>
  summarise(n = n()) |>
  mutate(geometry = cell_to_polygon(h3)) |>
  st_as_sf()
```

---

## Principles

- **Stream, don't download.** Use STAC + gdalcubes for raster; remote URLs + duckdbfs for vector. Download only when a local cache is explicitly needed.
- **Filter before collect.** With duckdbfs, always filter/select before calling `collect()` or `to_sf()`.
- **Cloud-optimized formats.** Prefer COG (raster), FlatGeobuf `.fgb`, or GeoParquet (vector) over Shapefile or plain GeoJSON.
- **All maps use mapgl.** Static maps, interactive maps, rasters, vectors — always mapgl.
- **gdalcubes handles the hard parts.** Mosaicking, reprojection, compositing, and cloud masking happen inside the cube pipeline.

## What NOT to do

| Instead of...                          | Use...                              |
|----------------------------------------|-------------------------------------|
| `terra::rast()` on many files          | `gdalcubes::raster_cube()`          |
| `stars::read_stars()`                  | `gdalcubes::raster_cube()`          |
| `tmap::tm_shape()`                     | `mapgl::maplibre()`                 |
| `leaflet::leaflet()`                   | `mapgl::maplibre()`                 |
| `sf::read_sf()` on large remote data   | `duckdbfs::open_dataset()`          |
| Downloading full tiles before clipping | STAC query with bounding box filter |
