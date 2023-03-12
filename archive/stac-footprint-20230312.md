# The stactools Raster Footprint Utility

*stactools: a command line utility and Python library for STAC.*

Every STAC Item contains a geometry object that defines the geospatial extent of the data described by the Item. For raster data, the geometry object contains one or more polygons enclosing the raster pixels, i.e., the footprint of the raster. At first glance, creating this *raster footprint* seems a straightforward task. Raster data is inherently rectangular, so a polygon with a vertex at each corner of the raster grid should be sufficient. Using an ESA Worldcover tile example, we see that it is cleanly bounded by a simple four-vertex polygon.

- Image

Case closed, right? Not quite. We "cheated" in the above example. The native coordinate reference system (CRS) of the ESA Worldcover image is the geographic WGS84 (EPSG:4326) system. This is the same CRS required by the geometry object in a STAC Item, meaning the raster data and our minimal footprint polygon naturally align in this case. But most remote sensing data is stored in a projected CRS and therefore distorted with respect to WGS84. A better example is a MODIS raster tile, which uses a sinusoidal projection. When viewed in WGS84, we see that a raster footprint defined by the four raster corners isn't up to the task.

- Image

We have a gap on the west and gore on the east. Not great. And it's not just aesthetics at play here. These inaccuracies are relevant to downstream applications that work with STAC Items, e.g., STAC APIs that respond to queries for Items within a defined polygon or box. Gaps and gores between Item geometry and the asset data will result in spurious and missing Items in the API response.

In addition to tightly bounding the raster grid extents in the WGS84 CRS, it may be desirable to exclude regions of the raster that do not contain meaningful information, e.g., open ocean in a land surface temperature product. Raster pixels in these regions are assigned a constant value, typically termed a "nodata" or "fill" value. Ideally, we'd like our footprint to exclude these nodata regions.

Finally, we want to balance our desire for high accuracy with the number of vertices in our raster footprint. Perfect accuracy can require (up to) a vertex at every pixel corner, resulting in huge lists of vertex coordinates in the polygon definition. When dealing with millions of STAC Items, large footprints lead to storage bloat and risk impacting spatial query performance. So we want to provide users flexibility to make their own decision in this trade-off.

>Our task, then, is to generate footprint polygons in the WGS84 CRS that tightly bound the valid raster pixels, regardless of the native raster CRS, with an eye on balancing footprint accuracy versus complexity.

## The stactools approach

The stactools approach is implemented in the `RasterFootprint` class, with the desired GeoJSON geometry object produced by the `footprint()` method. Given a `numpy` array of raster data, the minimum signature to produce a footprint is:

```python
footprint = RasterFootprint(numpy_data_array).footprint()
```

In practice, users are more likely to use one of the alternative constructors, e.g., the `from_href()` constructor:

```python
footprint = RasterFootprint.from_href(raster_file_path).footprint()
```

Regardless of the instantiation method, the `footprint()` method is orchestrating a four-step process:

1. Extract the raster footprint in the native CRS.
2. Densify the footprint with additional vertices in the native CRS.
3. Project the densified footprint to the WGS84 CRS.
4. Simplify the projected WGS84 footprint.

Let's work through these steps and the options available to tune the ultimate geometry characteristics.

### Shape extraction

Shape extraction uses rasterio's ...

Let's address what may be the most obvious approach right out of the gate: reproject the raster to WGS84, extract a footprint outlining the data, and apply some type of simplification to the footprint. This is, indeed, a valid approach and readily accomplished with the rasterio and shapely libraries. It is not, however, the stactools approach, for a few reasons. First, raster reprojection is computationally expensive. Second, in keeping with the desire for simple footprints, the stactools approach uses a convex hull to produce a single polygon footprint in instances where shape extraction produces multiple polygons (think of a coastline with many islands). Referring back to the MODIS example image, we can see that a convex hull applied in the WGS84 CRS will result in a gap on the east side, which is something we are deliberately trying to avoid. So the stactools utility takes a different approach:

### Densification

### Reprojection

### Simplification

## Final thoughts

Note this allows us to model concavities induced by reprojection. This is important, as concavity is not near as simple as convexity....

spatial extent with geographic coordinates in decimal degrees in the WGS84 coordinate reference system.

When creating an Item that describes a raster image, a spatial footprint of the image data must be generated to use for the Item geometry field. We intentionally use the imprecise term "footprint" here, as we are not trying to create a precise vector representation of the data of an image (which could contain as many polygons as pixels in the image!), but we are trying to do better than just a bounding box of the entire image, both data and no data. By footprint, we mean a vector that balances many competing factors, and generally represents a shape covering the data in an image. Figuring out the right parameter values to the methods in the raster_footprint module will usually require some experimental validation, to find exactly the right values for your data, CRS, and use cases.

There are a few properties we’d like to have in a footprint:

The footprint only covers the area on the image only where there is data.

The footprint has enough points to accurately capture the shape of the data in the EPSG:4326 CRS used by GeoJSON.

The footprint does not have too many points, such that it is becomes very large.

Discussion of these properties follows.

## Challenges

1. raster data is usually projected, that is, not in the WGS84 CRS required by the GeoJSON specification.
2. raster data may include areas (regions of pixels) that contain no data, that is, pixels with a value specified as "nodata" in the image metadata. In some cases we would like to exclude these nodata areas from our Item geometry.

Along the way, we'll discuss methods for balancing the need for accurately capturing the shape of the data while constraining the number of points in bounding polygon

## Caveat

We are not handling concavity. If we were, an alternative approach would be to reproject the data to WGS84 before extracting the shapes.

## Final Thoughts

An alternative approach is to reproject the data first and then define the boundaries. This would eliminate the densification tuning. This is a valid approach and was investigated when building the raster footprint utility. Ultimately, the goal of limiting complexity by using a convex hull steered us away from that approach. Referring to the MODIS example in this post, a convex hull in the WGS84 will consistently produce gaps...
