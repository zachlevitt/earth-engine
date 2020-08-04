![](header.png "Image classification in Earth Engine")

## Image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5 and [**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

### Introduction

This tutorial will introduce a workflow for utilizing topographic, spectral and spatial variables to classifying remotely-sensed imagery using machine learning approaches in Google Earth Engine (GEE). Using a combination of pre-loaded GEE satellite imagery and analysis methods, as well as topographic variables derived from uploaded LiDAR data, we will outline a novel image classification methodology. We will apply these tools to classify vegetation on the Channel Islands of California, though the tools are applicable to a wide range of use cases. We will utilize both supervised and unsupervised classification methods to show various methods for improving the accuracy and efficiency of the workflow.

***Note regarding code structure***

In order to increase the usability of our code, we will use modules to define functions that can be applied to any case study. You can read more about modules [here](https://medium.com/google-earth/making-it-easier-to-reuse-code-with-earth-engine-script-modules-2e93f49abb13). 

The following tutorial will reference module functions defined [here](https://code.earthengine.google.com/9ef0eb7a802163ba97e51a94a754379d). To connect to the module functions, add this line at the top of your script:

```javascript
var ct = require('users/zlevitt/chis:chisTools.js');
```

The code snippets below will highlight the module function as well as the corresponding script code that calls the function. To follow along, you can either reference these functions in your script, or create your own module script. Click [**here**](https://code.earthengine.google.com/ba0f64848eddfbce06369aa8cdbe21be) to view the completed script applied to vegetation on Santa Cruz Island.

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
		2. 
4. **Evaluate and visualize results**
	1. Confusion matrix
	2. Variable importance
	3. Visualization parameters
	4. Export data?

### 1. Upload and/or import data

To perform image classification with topographic variables, there are two necessary inputs:

1. **Satellite imagery** - National Agricultural Imagery Program (NAIP) or Sentinel
	* NAIP imagery
	* Sentinel imagery:
	 ```javascript
	 var sentinel = ct.loadSentinel(geometry,startDate,endDate,cloudPercentage);
	 ```
2. **Elevation data** - LiDAR, Shuttle Radar Topography Mission (SRTM), USGS National Elevation Dataset (NED)

In this tutorial, we utilize satellite imagery from Earth Engine and upload our own elevation data from LiDAR, which is not included in Earth Engine. 

If you have available LiDAR data for your study region, you can upload a GeoTIFF as an asset to Earth Engine by clicking on the **Assets** tab, then the **New** button, and upload your GeoTIFF. 

If not, you can use the USGS NED for study areas within the United States or SRTM data for global case studies. Add these to your script by searching for one of these datasets in the search bar and clicking the **Import** button. It should appear at the top of your script under an **Imports** header and you can change the name to whatever you like. In this script, we will name our elevation data **```dem```**.



* We will reproject to ....
* Define variables like number of classes, bands, scale, etc
* projection if naip, or sentinel 
* cloudy
* If you have never used GEE before, here is a helpful place to start [https://jeffhowarth.github.io/eeprimer/start/getGEE/](https://jeffhowarth.github.io/eeprimer/start/getGEE/)
* Our code will be broken down into two main files - modules and function calls that bring it all together



### Data pre-processing 

* NAIP - add NDVI, add year, etc
* Sentinel - cloudy
* Elevation data - we used ArcMap to gather and clean LiDAR data, if this is not available you could use SRTM or DEM USGS data. But we wanted 1.5 m data
* 

### Modules

### Topographic variables

We calculated several topographic variables to include in our model:

##### **Slope**
```javascript
//Module function
exports.calculateSlopeDegrees = function(dem,mask){
	return ee.Terrain.slope(dem).mask(mask)
}

//Script
var slopeDegrees = ct.calculateSlopeDegrees(maskedDEM,mask);
```
##### **Heat load index** (from Theobald et al, 2016)
##### **Topographic Position Index**
##### **Mean Topographic Position Index**
##### **Canopy Height Model** (Digital Surface Model - Digital Elevation Model)



#### Spectral variables

### Within vs. Outside of GEE

### Classification methods

### Supervised
* Training
* Validation
1. confusion matrix
2. producers and consumers accuracy
* Test in a study area

### Unsupervised

### Visualize results

### Export results

<!-- ##### Define variables

The first step is to define 

```javascript
var outScale = 1.5;
var outCRS = 'EPSG:26911';
var year = 2018;
``` -->

<!-- ### Background

There are two primary motiviations for this work:

1. Apply machine learning approaches to identify vegetation classes using topographic, spectral and spatial variables.
2. Update vegetation data for the Channel Islands to aid conservation and environmental projects (Last updated in [2007](http://iws.org/CISProceedings/7th_CIS_Proceedings/Cohen_et_al.pdf) for [Santa Cruz Island](https://map.dfg.ca.gov/metadata/ds0563.html), the largest of the Channel Islands).
 -->




