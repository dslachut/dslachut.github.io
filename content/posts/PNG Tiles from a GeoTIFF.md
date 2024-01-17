---
title: PNG Tiles from a GeoTIFF
date: 2024-01-17T10:36:52-06:00
draft: true
tags:
  - how-to
categories:
  - GIS
  - '"raster data"'
keywords: 
publishDate: 2024-01-17T10:36:52-06:00
toc: true
---
This is a quick look at how to build a PNG tileset from a raster data set.

Get some data
---
First, we need a data set. You can find the USDA Cropland Data Layer [here](https://www.nass.usda.gov/Research_and_Science/Cropland/Release).  Or go here to directly pull the 2022 data:
```
https://www.nass.usda.gov/Research_and_Science/Cropland/Release/datasets/2022_30m_cdls.zip
```
Download and unzip that file to get  `2022_30m_cdls.tif`.

This is a GeoTIFF file covering the continental United States in an equal area projection (EPSG 5070). Each pixel represents an square, 30m on a side. The equal area projection makes it easy to do zonal statistics by counting pixels. Each pixel is coded with a value between zero and 255 (inclusive). This value represents one of several recognized land covers including crop type, deciduous vs coniferous forest, and developed area. The TIF also has a colormap with each of these pixel values mapped to a specific RGBA color.

This is not a *cloud optimized* GeoTIFF. If you're going to be doing a lot of on-the-fly masking and cropping, you will want to convert it to a COG. But, since this is just about making a tileset, we don't need to cover that part.

Translate to VRT
---
Once we have the data, we need to translate it into a VRT to expose the color mapping for the next step.
``` bash
gdal_translate \
-q \
-strict \
-of vrt \
-expand rgba
2022_30m_cdls.tif
tmp0.vrt
```
To explain the options:
- `-q` runs the command in quiet mode. Keep this if you're running it in an automated process or a pipeline. Drop this flag if you're running it interactively, debugging, or don't mind cluttered output.
- `-strict` runs the command in strict mode when translating to the output format. This one shouldn't matter, but in my case I want to make sure the pipeline breaks noisily if I feed it bad data.
- `-of vrt` explicitly declares the VRT output format
- `-expand rgba` is the purpose of this processing step. It combines the GeoTIFF pixel values and colormap to create a virtual 3-color (with transparency) image.

Build the tiles
---
This step converts the GeoTIFF, via the VRT, into PNG tiles.
``` bash
gdal2tiles.py \
-q \
--resume \
--exclude \
--s_srs=epsg5070 \
--profile=mercator \
--resampling=near \
--zoom=12 \
--processes=16 \
--tilesize=256 \
--webviewer=none \
--srcnodata=0,0,0,0 \
--title=CDL_2022 \
2022
```
- `-q` as above
- `--resume` generates only missing files. If nothing breaks, this option does nothing. If you're running this manually and something does break along the way, it will save you some time not regenerating tiles you already created.
- `--exclude` saves time and storage by skipping entirely transparent tiles---missing data or no data---that just happen to lie within the bounding box of the GeoTIFF/VRT.
- `-s_srs=epsg5070` explicitly specifies the coordinate system of the source data. It's better to be explicit with this where you can, because we are converting between reference systems.
- `--profile=mercator` specifies that our output tiles will be in Web Mercator projection, which is standard for Mapbox, Leaflet, and Google Maps
- `--resampling=near` because we're changing map projections. This process requires resampling the raster. In this case, we are dealing with categorical values---e.g. "corn", "soybeans", "open water", etc. We do not want any smoothing, so we choose nearest-neighbor resampling.
- `--zoom=12` sets the zoom level(s) at which tiles are generated. This can be a single value or a range and is generally chosen by trial and error.
- `--processes=16` sets the number of processor cores to use for generating tiles. Set this based on the machine you're using. This task is highly parallel, so push it as high as your machine will allow.
- `--tilesize=256` sets each output PNG file to have 256 pixels on a side. This is the size Mapbox prefers other map renderers may prefer 512 or something else.
- `--webviewer=none` prevents some extraneous output. By default, `gdal2tiles` generates static web pages to demonstrate your tiles on a map. If you already have a map you like, you don't need those.
- `--srcnodata=0,0,0,0` sets pixels as colorless and transparent where the source file has no data. In the present case, the CDL GeoTIFF provides no data for the Gulf of Mexico, as it is within the bounding box of CONUS, but no crops grow there. This sets that area as transparent so that the `exclude` option can drop those areas from the tileset.
- `--title=CDL_2022` sets the title of the tileset in the tileset's metadata
- `2022` is the output folder where the generated tiles will be written.
The output data will then be in a folder like `./2022/{z}/{x}/{y}.png`:  `2022` being the output folder, the only `{z}` being our `--zoom` value of `12`, and `{x}` and `{y}` being the web mercator tile coordinates for the given zoom level.

Here, you can upload the tiles to a server or object storage to serve.
