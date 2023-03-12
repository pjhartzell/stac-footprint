# The `stactools` Raster Footprint Utility

The geometry field in a STAC Item defines the spatial extent of the Item's asset data using point, line, and polygon structures. Beyond just good hygiene, accurate Item geometries are important when searching against spatially-aware databases for Items that intersect a given area of interest. Incorrect or inaccurate Item geometries will produce search results that contain Items with asset data that does not intersect the queried area of interest or, conversely, fail to return Items that do. This post looks at one approach to generating accurate STAC Item geometries for _raster_ data, while simultaneously limiting their complexity, and its implementation in the `stactools` raster footprint utility.

## Motivating example

At first blush, defining the spatial extent of a raster data product, e.g., a GeoTIFF, is straightforward. Raster data is inherently rectangular, so simply defining a polygon with a vertex at each of the four corners of the image should be sufficient. For example, look at this ESA Worldcover tile and its geometry (red polygon), which is just a polygon defined by the corners of the raster grid:

- Image

Easy! However, we "cheated" in the above example. The native coordinate reference system (CRS) of the example image is WGS84 (EPSG:4326), which is the same required by the STAC Item geometry object. Most remote sensing raster data is in a projected CRS, so let's look at an example tile of MODIS data where the native CRS is a sinusoidal projection. Connecting the four vertices of the raster grid does not look so good when viewed in WGS84:

- Image

We have a gap on the west and gore on the east. Not great. Note, also, the large region of nodata pixels in the raster; it may be desirable to create a polygon that exludes this region so Item searches only return Items where there is useful data.

So we see there are two challenges when creating polygons for raster data:

1. raster data is often projected, that is, not in the WGS84 CRS required by the GeoJSON specification. Distortion...
2. raster data may include areas (regions of pixels) that contain no data, that is, pixels with a value specified as "nodata" in the image metadata. In some cases we would like to exclude these nodata areas from our Item geometry.

Finally, a note on on our use of the imprecise term "footprint". This is intentional, as we are not trying to create a precise vector representation of the data of an image (which could contain as many polygons as pixels in the image!). Rather, but we are trying to do better than just a bounding box of the entire image, both data and no data. By footprint, we mean a vector that balances many competing factors, and generally represents a shape covering the data in an image. Figuring out the right parameter values to the methods in the raster_footprint module will usually require some experimental validation, to find exactly the right values for your data, CRS, and use cases.

We'll work through one approach to these challenges while also balancing concerns about number of vertices vs accuracy.

## Handling projection distortion

## Isolating valid data

## The stactools raster footprint utility

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
