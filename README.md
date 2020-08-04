![](header.png "Image classification in Earth Engine")

## Image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5 and [**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

### Introduction

This tutorial will introduce a workflow for classifying remotely-sensed imagery in Google Earth Engine (GEE) using machine learning models trained on topographic and spectral variables. Using a combination of pre-loaded GEE satellite imagery and analysis methods, as well as topographic variables derived from uploaded LiDAR data, we outline a novel image classification methodology. These methods are applied to a specific case study of classifying vegetation on the Channel Islands of California, though the tools are applicable to a wide range of use cases. We utilize both supervised and unsupervised classification methods to demonstrate various methods for improving the accuracy and efficiency of the workflow.

***Note regarding code structure***

In order to increase the usability of our code, we use modules to define functions that can be applied to any case study. You can read more about modules [here](https://medium.com/google-earth/making-it-easier-to-reuse-code-with-earth-engine-script-modules-2e93f49abb13). The following tutorial will reference module functions defined [here](https://code.earthengine.google.com/9ef0eb7a802163ba97e51a94a754379d). To connect to the module functions, add this line at the top of your script:

```javascript
var ct = require('users/zlevitt/chis:chisTools.js');
```

You can reference these functions in your script or modify them to fit your context and create your own module script. The code snippets below are from the [**completed script**](https://code.earthengine.google.com/ba0f64848eddfbce06369aa8cdbe21be), which applies these methods to classify vegetation on Santa Cruz Island.

If you have never used GEE before, here is a helpful place to start: [https://jeffhowarth.github.io/eeprimer/start/getGEE/](https://jeffhowarth.github.io/eeprimer/start/getGEE/)

### Overview

This tutorial will progress in four steps:

1. **Upload and/or import data**
	1. Satellite imagery
	2. Elevation data
2. **Calculate and combine variables**
	1. Spectral variables
	2. Topographic variables
	3. Combining spectral and topographic variables
3. **Set up and apply classification methods**
	* Supervised
		1. Create training and validation data with stratified sampling
		2. Join topographic and spectral variables
		3. Run Random Forest classification
	* Unsupervised?
		1. Segmentation?
		2. Other unsupervised methods?
4. **Evaluate and visualize results**
	1. Confusion matrix
	2. Variable importance
	3. Visualization parameters
	4. Export data?

### 1. Upload and/or import data

To perform image classification with topographic variables, there are two necessary inputs:

**Satellite imagery** - National Agricultural Imagery Program (NAIP) or Sentinel
```javascript
//Constants
var outScale = 10;
var outCRS = 'EPSG:32611'
var startDateWinter = '2019-02-15';
var endDateWinter = '2019-03-15';
var startDateSummer = '2019-08-01';
var endDateSummer = '2019-09-30';
var cloudPercentage = 0.2;

//We will utilize two time periods for Sentinel data
var sentinelWinter = ct.loadSentinel(extent,startDateWinter,endDateWinter,cloudPercentage);
var sentinelSummer = ct.loadSentinel(extent,startDateSummer,endDateSummer,cloudPercentage);
```

**Elevation data** - LiDAR, Shuttle Radar Topography Mission (SRTM), USGS National Elevation Dataset (NED)

In this tutorial, we our own elevation data from LiDAR, which is not included in Earth Engine. If you have available LiDAR data for your study region, you can upload GeoTIFFs as assets to Earth Engine by clicking on the **Assets** tab, then the **New** button, and upload your GeoTIFF. 

```javascript
var dem = ct.loadDEM()
    .reduceResolution({ // Force the next reprojection to aggregate instead of resampling.
      reducer: ee.Reducer.mean(),
      maxPixels: 1024
    }).reproject({ // Request the data at the scale and projection of the MODIS image.
      crs: outCRS,
      scale: outScale
});

var dsm = ct.loadDSM()
    .reduceResolution({
      reducer: ee.Reducer.mean(),
      maxPixels: 1024
    }).reproject({
      crs: outCRS,
      scale: outScale
});
```

If not, you can use the USGS NED for study areas within the United States or SRTM data for global case studies. Add these to your script by searching for one of these datasets in the search bar and clicking the **Import** button. It should appear at the top of your script under an **Imports** header and you can change the name to whatever you like. In this script, we will name our elevation data **```dem```**. You will not be able to include the ```dsm``` in your analysis.


* cloudy
* Our code will be broken down into two main files - modules and function calls that bring it all together



### 2.. Calculate and combine variables

* NAIP - add NDVI, add year, etc
* Sentinel - cloudy
* Elevation data - we used ArcMap to gather and clean LiDAR data, if this is not available you could use SRTM or DEM USGS data. But we wanted 1.5 m data
* 

#### Topographic variables

We calculated several topographic variables using our DEM and DSM:   
        
      
      //var dsm = dsm.resample('bilinear').reproject(outCRS,null,outScale).mask(mask);
      //print(dem.getInfo())
      var maskedDEM = ct.maskDEM(dem, mask);
      var maskedDSM = ct.maskDEM(dsm, mask);

##### Slope
```javascript
var slopeDegrees = ct.calculateSlopeDegrees(maskedDEM,mask);
```

##### Heat load index, based on Theobald et al (2015)
```javascript
var theobaldHLI = ct.calculateTheobaldHLI(maskedDEM,mask);
```

##### Mean Topographic Position Index
```javascript
var demMean_270m = ct.calculateNeighborhoodMean(maskedDEM,27)
var demStdDev_270m = ct.calculateNeighborhoodStdDev(maskedDEM,27)
var stdTPI_270m = ct.calculateStandardizedTPI(maskedDEM,demMean_270m,demStdDev_270m)

var demMean_810m = ct.calculateNeighborhoodMean(maskedDEM,81)
var demStdDev_810m = ct.calculateNeighborhoodStdDev(maskedDEM,81)
var stdTPI_810m = ct.calculateStandardizedTPI(maskedDEM,demMean_810m,demStdDev_810m)

var demMean_2430m = ct.calculateNeighborhoodMean(maskedDEM,243)
var demStdDev_2430m = ct.calculateNeighborhoodStdDev(maskedDEM,243)
var stdTPI_2430m = ct.calculateStandardizedTPI(maskedDEM,demMean_2430m,demStdDev_2430m)

var meanTPI = ct.calculateMeanTPI(stdTPI_270m,stdTPI_810m,stdTPI_2430m)
```

##### Topographic Position Index, based on Theobald et al (2015)
```javascript
var tpi_270m = ct.calculateTPI(maskedDEM,demMean_270m)
```

##### Canopy Height Model (Digital Surface Model - Digital Elevation Model)
```javascript
var difference = ct.elevationDifference(maskedDEM,maskedDSM)
	.reproject(outCRS,null,outScale)
	.resample('bilinear')
```

#### Spectral variables

##### NDVI Difference
##### Forest-cropland
##### NDVI 45
##### Burn ratio


### 3. Set up and apply classification methods


### Supervised
* Training
* Validation
1. confusion matrix
2. producers and consumers accuracy
* Test in a study area

### Unsupervised

### 4. Evaluate and visualize results

<!-- ##### Define variables

The first step is to define 

```javascript

``` -->

<!-- ### Background

There are two primary motiviations for this work:

1. Apply machine learning approaches to identify vegetation classes using topographic, spectral and spatial variables.
2. Update vegetation data for the Channel Islands to aid conservation and environmental projects (Last updated in [2007](http://iws.org/CISProceedings/7th_CIS_Proceedings/Cohen_et_al.pdf) for [Santa Cruz Island](https://map.dfg.ca.gov/metadata/ds0563.html), the largest of the Channel Islands).
 -->




