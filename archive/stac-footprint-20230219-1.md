# Defining Raster Footprints for STAC Item Geometries

Per the STAC specification, A STAC Item "...is simply a GeoJSON Feature with a well-defined set of additional attributes ('foreign members')". As such, a STAC Item - hereafter referred to as simply an "Item" - must contain a Geometry object with geographic coordinates in decimal degrees in the WGS84 coordinate reference system. Defining the content of the geometry field for raster data is the focus of this post. For context, here is a minimal Item:

- maybe a geojson.io screenshot here

```json
{
    "type": "Feature",
    "geometry": "",
    "id": "example-stac-item"
}
```

When creating an Item that describes a raster image, a spatial footprint of the image data must be generated to use for the Item geometry field. We intentionally use the imprecise term "footprint" here, as we are not trying to create a precise vector representation of the data of an image (which could contain as many polygons as pixels in the image!), but we are trying to do better than just a bounding box of the entire image, both data and no data. By footprint, we mean a vector that balances many competing factors, and generally represents a shape covering the data in an image. Figuring out the right parameter values to the methods in the raster_footprint module will usually require some experimental validation, to find exactly the right values for your data, CRS, and use cases.

There are a few properties we’d like to have in a footprint:

The footprint only covers the area on the image only where there is data.

The footprint has enough points to accurately capture the shape of the data in the EPSG:4326 CRS used by GeoJSON.

The footprint does not have too many points, such that it is becomes very large.

Discussion of these properties follows.

There a few challenges as well:

- raster data is usually projected, that is, not in the WGS84 CRS required by the GeoJSON specification.
- raster data may include areas (regions of pixels) that contain no data, that is, pixels with a value specified as "nodata" in the image metadata. In some cases we would like to exclude these nodata areas from our Item geometry.

We will work with an example raster image with both of these characteristics. The image CRS is the MODIS Sinusoidal projection and contains a large area of nodata.

- image here
