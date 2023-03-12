# Defining Raster Footprints for STAC Item Geometries

STAC Item geometries define the spatial extent of the data being described. Beyond just good hygiene, accurate Item geometry fields are critical for searching spatially-aware databases for Items that intersect a given area of interest. At first blush, defining the spatial extent for a raster data product is straightforward. Raster data is inherently rectangular, so simply defining a polygon with a vertex at each of the four corners of the image should be sufficient. For example, look at this ESA Worldcover tile and it's geometry:

- Image

Accurate bounds in STAC Item geometries are important for returning correct Item searches. At first look, defining the polygon bounds for a STAC Item is straightforward. Simply place a point at each of the four corners:

- Image

However, we "cheated" in the above example. The native coordinate reference system (CRS) of the example image is WGS84 (EPSG:4326), which is the same required in STAC Item geometry object (a STAC Item is a GeoJSON Feature, which uses the geographic WGS84 CRS). Now let's take a look at a MODIS tile of data where the native CRS is a custom sinusoidal projection. Connecting the four vertices of the pixel grid does not look so good when viewed in WGS84:

- Image

We have a gap on the west and gore on the east. The gap will cause Item searches, e.g., with pystac-client, to return the Item for this raster asset where there actually is no coverage. Conversely, the gore will result in missing Items.

Simply projecting the four corners from the native CRS to WGS84 results in a polygon with both a gap and a gore relative to the data. Note, also, there is a large region of nodata pixels in the raster, and it may be desirable to create a polygon that exludes this region.

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
