![](header.png "Calculating topographic variables in Earth Engine")

# Calculating topographic variables from Theobald et al. (2015) in Google Earth Engine

## Introduction

This tutorial outlines a workflow for utilizing module functions in Google Earth Engine (GEE) to calculate a set of topographic variables derived from Theobald et al. (2015). We use modules to define functions that can be applied to any case study. GEE [Modules](https://medium.com/google-earth/making-it-easier-to-reuse-code-with-earth-engine-script-modules-2e93f49abb13) are helpful for producing reusable code and allow anyone to reference pre-defined functions with their own inputs. You can reference these functions in your script or modify them to fit your context and create your own module script. This [tutorial](./modules.md) offers descriptions of the module functions. To connect to the module functions defined [here](https://code.earthengine.google.com/9ef0eb7a802163ba97e51a94a754379d), add this line at the top of your script:

```javascript
var tt = require('users/zlevitt/chis:topoTools.js');
```

Many of these methods are used in an example app that allows user to upload a DEM and automatically calculate topographic variables. This app can be found [here](#https://code.earthengine.google.com/1bc8668dd8093ecff8871e6cc64bcb17). The goal of this tutorial is to allow users to understand the code and alter it to fit a particular use case. 

## What are the aspects of this tutorial?

1. [**Upload or import DEM**](#upload-or-import-dem)
2. [**Clip DEM to study region**](#clip-dem-to-study-region)
	1. [Draw region](#draw-region)
	2. [Vector data](#vector-data)
	3. [DEM extent](#dem-extent)
3. [**Calculate topographic variables**](#calculate-topographic-variables)
	* [Topographic variables](#topographic-variables)
		* [Heat load index](#heat-load-index)
		* [Topographic position index](#topographic-position-index) (TPI)
		* [Mean TPI](#mean-tpi)
		* [Slope](#slope)
		* [Landforms](#landforms)
4. [**Visualize results**](#visualize-results)
	* [All classes](#all-classes)
	* [Simplified classes](#simplified-classes)

## Upload or import DEM

There are several options for elevation data in GEE, including Shuttle Radar Topography Mission (SRTM) and USGS National Elevation Dataset (NED). These options offer national (USGS NED) or global (SRTM) elevation rasters. This script will also work with uploaded DEM data like LiDAR. 

In this tutorial, we use our own elevation data from LiDAR, which is not included in Earth Engine and offers both elevation and surface models for certain regions. If you have available LiDAR data for your study region, you can upload GeoTIFFs as assets to Earth Engine by clicking on the **Assets** tab, then the **New** button, and upload your GeoTIFF. The following code shows how to utilize pre-defined modules to load a LiDAR DEM, process the DEM, and then calculate topographic variables.

Use the following code to load an image asset and define the coordinate reference system and scale.

```javascript
//Load image asset 
var dem = tt.loadImageAsset('users/zlevitt/chis_assets/sci_dem_1p5m');
var CRS = dem.projection().crs();
var elevationScale = dem.projection().nominalScale();
```
[View **loadImageAsset** function description](./modules.md#loadImageAsset) </br>
[View **calculateMask** function description](./modules.md#calculatemask)


## Clip DEM to study region
Depending on the extent of your DEM, you may want to clip your DEM before outputting the topographic variables. There are three choices for this step: clip to a drawn region, vector data, or the extent of the DEM. 

#### Draw region
If you want to draw a region on the map, use the 'Draw a rectangle' to create a geometry on the map. Then use this code to load it as the clipArea variable.
```javascript
var clipArea = map.drawingTools().layers().get(0).toGeometry();
``` 

#### Vector data
Paste your desired vector data as the path to load this data as your clip area.
```javascript
var clipArea = ee.FeatureCollection('users/zlevitt/chis_assets/SCI_sampleArea').geometry();
``` 

#### DEM extent
Uploaded DEMs will most likely have a geometry attribute that you can reference to clip to the extent of the DEM.
```javascript
var clipArea = dem.geometry();
```

Regardless of your method, use the following code to clip the DEM:
```javascript
var demClipped = dem.clip(clipArea);
```


## Calculate topographic variables

Once you have successfully loaded and processed the elevation data, you can begin to calculate topographic variables. 

#### Slope
The first topographic variable is slope, which helps distinguish between different landform types and will be an input to our heat load index.
```javascript
var slopeDegrees = tt.calculateSlopeDegrees(demClipped);
```
[View **calculateSlopeDegrees** function description](./modules.md#calculateslopedegrees)

#### Topographic Position Index
Based on Theobald et al (2015), we calculate a multi-scale topographic position index (TPI) that measures relative topographic relief. This is measured by subtracting mean elevation for a neighborhood of cells from the base elevation data. In this case, we calculate TPI with kernel radius of 270m.
```javascript
var demMean_270m = tt.calculateNeighborhoodMean(demClipped,180);
var tpi_270m = tt.calculateTPI(demClipped,demMean_270m);
```
[View **calculateTPI** function description](./modules.md#calculatetpi)

#### Mean TPI
Mean TPI is a helpful factor to be able to distinguish between landform types and requires only a DEM. This workflow is based on Theobald et al's (2015) method for calculating a mean TPI using three resolutions. We calculate the standardized TPI, which is the topographic position index divided by the standard deviation of elevation at the same spatial resolution. These standardized TPIs are then averaged to compute a Mean TPI layer.
```javascript
var demMean_270m = tt.calculateNeighborhoodMean(demClipped,180);
var demStdDev_270m = tt.calculateNeighborhoodStdDev(demClipped,180);
var stdTPI_270m = tt.calculateStandardizedTPI(demClipped,demMean_270m,demStdDev_270m);
  
var resampledDEM_810m = tt.processElevationData(demClipped,CRS,elevationScale.multiply(3),null)
var demMean_810m = tt.calculateNeighborhoodMean(demClipped,180);
var demStdDev_810m = tt.calculateNeighborhoodStdDev(demClipped,180);
var stdTPI_810m = tt.calculateStandardizedTPI(demClipped,demMean_810m,demStdDev_810m);
      
var resampledDEM_2430m = tt.processElevationData(demClipped,CRS,elevationScale.multiply(9),null)
var demMean_2430m = tt.calculateNeighborhoodMean(demClipped,180);
var demStdDev_2430m = tt.calculateNeighborhoodStdDev(demClipped,180);
var stdTPI_2430m = tt.calculateStandardizedTPI(demClipped,demMean_2430m,demStdDev_2430m);
        
var meanTPI = tt.calculateMeanTPI(stdTPI_270m,stdTPI_810m,stdTPI_2430m);
```
[View **calculateMeanTPI** function description](./modules.md#calculatemeantpi)</br>
[View **calculateStandardizedTPI** function description](./modules.md#calculatestandardizedtpi)</br>
[View **calculateNeighborhoodStdDev** function description](./modules.md#calculateneighborhoodstddev)</br>
[View **calculateNeighborhoodMean** function description](./modules.md#calculateneighborhoodmean)</br>
[View **processElevationData** function description](./modules.md#processelevationdata)</br>


#### Heat load index
We based our heat load index (HLI) on a workflow modified by Theobald et al (2015). This continuous heat-insolation index attempts to measure the amount of heat exposure on a given piece of land based on topography. 
```javascript
var theobaldHLI = tt.calculateTheobaldHLI(demClipped);
```
[View **theobaldHLI** function description](./modules.md#theobaldhli)


#### Landforms
Our landforms are based on the classifications developed by Theobald et al. (2015).
```javascript
var landformsTheobald = tt.calculateLandforms(demClipped,slopeDegrees,theobaldHLI,meanTPI,tpi_270m);
```
[View **calculateLandforms** function description](./modules.md#calculatelandforms)


## Visualize results
There are two primary ways to visualize the landform classes. First, we can visualize all of the classes, including distinctions between warm, neutral, and cool for each land class. 
Alternatively, we can also visualize simplified classes, which include valley, lower slope, upper slope, and summit. The code below remaps the landforms to allow for visualization and the creation of legends, and then outputs charts showing the percent of the output region comprised by each class.


### All classes

```javascript
//Remap landforms to fit with visualization parameters
var remappedLandforms = tt.remapLandforms(landformsTheobald);

//Class dictionary for full landform file
var landformsDict = tt.landformsDict;

//Create chart for all classes
var chartAll = tt.outputChart(remappedLandforms,clipArea,50,landformsDict)

print('Hover over the charts to view information about each class.')
print(chartAll.setOptions(tt.makeChartStyle('Breakdown of output region by landform class', tt.landformsPalette)))

Map.addLayer(remappedLandforms,{min:0,max:14,palette:tt.landformsPaletteDisplay},'landforms - all')
Map.centerObject(landformsTheobald,12)
```
[View **remapLandforms** function description](./modules.md#remaplandforms)</br>
[View **outputChart** function description](./modules.md#outputchart)</br>
[View **makeChartStyle** function description](./modules.md#makechartstyle)

### Simplified classes

```javascript
//Remap landforms to fit with visualization parameters
var remappedLandformsSimple = tt.remapLandformsSimple(landformsTheobald);

//Class dictionary for simplified
var landformsSimpleDict = tt.landformsSimpleDict;

//Load landform pallette for simple legend 
var valuesSimple = tt.landformsSimplePalette;

//Create chart for all classes
var chartSimple = tt.outputChart(remappedLandformsSimple,clipArea,50,landformsSimpleDict)

print('Hover over the charts to view information about each class.')
print(chartSimple.setOptions(tt.makeChartStyle('Breakdown of output region by landform class', tt.landformsSimplePalette)))


Map.addLayer(remappedLandformsSimple,{min:0,max:3,palette:tt.landformsSimplePaletteDisplay},'landforms - simplified')
Map.centerObject(demClipped,12)
```
[View **remapLandformsSimple** function description](./modules.md#remaplandformssimple)</br>
[View **outputChart** function description](./modules.md#outputchart)</br>
[View **makeChartStyle** function description](./modules.md#makechartstyle)

