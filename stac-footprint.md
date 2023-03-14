# The stactools Raster Footprint Utility

*stactools: a command line utility and Python library for STAC.*

Every STAC Item contains a geometry object that defines the geospatial extent of the data described by the Item. For raster data, the geometry object contains one or more polygons enclosing the raster pixels, i.e., the footprint of the raster. At first glance, creating this *raster footprint* seems a straightforward task. Raster data is inherently rectangular, so a polygon with a vertex at each corner of the raster grid should be sufficient. Using an ESA Worldcover tile as an example, we see that it is cleanly bounded by a simple four-vertex polygon.

- Image

Case closed, right? Not quite. We "cheated" in the above example. The native coordinate reference system (CRS) of the ESA Worldcover image is the geographic WGS84 (EPSG:4326) system. This is the same CRS required by the geometry object in a STAC Item, meaning the raster data and our minimal footprint polygon naturally align in this case. But most remote sensing data is stored in a projected CRS and is therefore distorted with respect to WGS84. A better example is a MODIS raster tile, which uses a sinusoidal projection. When viewed in WGS84, we see that a raster footprint defined by the four raster corners isn't up to the task.

- Image

We have a gap on the west and gore on the east. Not great. And it's not just aesthetics at play here. These inaccuracies are relevant to downstream applications that work with STAC Items, e.g., STAC APIs that respond to queries for Items within a defined polygon or bounding box. Discrepancies between Item geometry objects and actual data extents will result in spurious or missing Items in API responses.

In addition to tightly bounding the raster grid extents in the WGS84 CRS, it may be desirable to exclude regions of the raster that do not contain meaningful information, e.g., open ocean in a land surface temperature product. Raster pixels in these regions are assigned a constant value, typically termed a "nodata" or "fill" value. We'd like an option to exclude these "nodata" regions from a raster footprint.

Finally, we want to balance footprint accuracy versus complexity (size). Perfect accuracy can require (up to) a vertex at every pixel corner, resulting in polygons with huge lists of vertex coordinates. When dealing with millions of STAC Items, large footprints cause storage bloat and risk impacting spatial query performance when retrieving Items from a database. So we'd like to have options to tune this tradeoff between accuracy and complexity.

>The task, then, is to generate raster footprint polygons in the WGS84 CRS that tightly bound valid raster pixels, regardless of the native raster CRS, with an eye on balancing footprint accuracy versus complexity.

## Approach

`stactools` takes a five-step approach to raster footprint creation:

1. Create a mask of valid data pixels from the raster data.
2. Extract the footprint of the valid data in the raster's native CRS.
3. Densify the footprint with additional vertices in the raster's native CRS.
4. Project the densified footprint to the WGS84 CRS.
5. Simplify the projected WGS84 footprint.

These five steps are executed by the `footprint()` method of the `RasterFootprint` class. The minimal signature to create a raster footprint requires a `numpy` array of raster data, the native raster CRS, and the raster transform matrix:

```python
footprint = RasterFootprint(data_array, crs, transform).footprint()
```

In practice, users are more likely to employ one of the `RasterFootprint` alternative constructors to create a class instance from a file path or STAC Item, bypassing the need to manually provide the raster data array, CRS, and transform. These constructors are reviewed in the "Implementation" section toward the end of this blog post. For now, let's work through the five step approach and the options that are available to tune the footprint characteristics. We'll use the same MODIS tile as above as our example and produce images for each option to illustrate the impact.

### Mask

The first step is to create a 2D mask array of valid data pixel locations in the given `data_array`: "nodata" value pixels are set to 0 and valid data pixels are set to 1. If the raster data is multi-band, meaning the given `data_array` and initial mask array are 3D, the initial mask array is flattened to 2D along the band axis. The existence of valid data in any band will cause the 2D mask array at that pixel to be set to 1, i.e., the mask is flattened using `OR` logic. For a pixel in the 2D mask array to be set to 0, the corresponding pixel in each band must contain the "nodata" value.

Two options apply to the mask creation step:

1. `no_data`: Pixel value to exclude from the raster footprint. Defaults to `None`, in which case a footprint for the entire raster grid is calculated.
2. `bands`: List of band indices to include in the raster footprint calculation. Defaults to `[1]`, in which case only the first band is used. *This option is only available in the alternative constructors, where it is used to generate the `data_array` argument.*

```python
footprint = RasterFootprint(data_array, crs, transform, no_data=0).footprint()
```

### Shape extraction

Shape extraction uses rasterio's `features.shapes` function

convex hull

### Densification

### Reprojection

### Simplification

## Implementation

## Closing thoughts

Let's address what may be the most obvious approach right out of the gate: reproject the raster to WGS84, extract a footprint outlining the data, and apply some type of simplification to the footprint. This is, indeed, a valid approach and readily accomplished with the rasterio and shapely libraries. It is not, however, the stactools approach, for a few reasons. First, raster reprojection is computationally expensive. Second, in keeping with the desire for simple footprints, the stactools approach uses a convex hull to produce a single polygon footprint in instances where shape extraction produces multiple polygons (think of a coastline with many islands). Referring back to the MODIS example image, we can see that a convex hull applied in the WGS84 CRS will result in a gap on the east side, which is something we are deliberately trying to avoid. So the stactools utility takes a different approach:
Note this allows us to model concavities induced by reprojection. This is important, as concavity is not near as simple as convexity....

An alternative approach is to reproject the data first and then define the boundaries. This would eliminate the densification tuning. This is a valid approach and was investigated when building the raster footprint utility. Ultimately, the goal of limiting complexity by using a convex hull steered us away from that approach. Referring to the MODIS example in this post, a convex hull in the WGS84 will consistently produce gaps...

Improvements:

- spatial extent with geographic coordinates in decimal degrees in the WGS84 coordinate reference system.
- Unable to force nodata=None to produce raster grid footprint via alternative constructors
- We can OR bands, but not Assets
