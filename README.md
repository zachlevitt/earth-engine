![](header.png "Image classification in Earth Engine")

## Image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5</br>
[**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

### Introduction

This tutorial outlines a workflow for classifying remotely-sensed imagery in Google Earth Engine (GEE) using machine learning models trained on topographic and spectral variables. Using a combination of pre-loaded GEE satellite imagery and analysis methods, as well as topographic variables, we describe the steps to implement an image classification methodology in GEE. We utilize supervised classification methods and demonstrate various methods for improving the accuracy and efficiency of the workflow. While we apply these methods to classify vegetation on the Channel Islands of California, our module functions and the general workflow are applicable to a wide range of use cases. 

<!-- The tutorial has been broken down into major tasks that you can pick and choose to fit your particular use case. 

***Note regarding code structure*** -->

In order to increase the usability of our code, we use modules to define functions that can be applied to any case study. GEE [Modules](https://medium.com/google-earth/making-it-easier-to-reuse-code-with-earth-engine-script-modules-2e93f49abb13) are helpful for producing reusable code and allow anyone to reference pre-defined functions with their own inputs. You can reference these functions in your script or modify them to fit your context and create your own module script. This [tutorial](./modules.md) offers descriptions of the module functions. To connect to the module functions defined [here](https://code.earthengine.google.com/9ef0eb7a802163ba97e51a94a754379d), add this line at the top of your script:

```javascript
var ct = require('users/zlevitt/chis:chisTools.js');
```

The code snippets below are from the [**completed script**](https://code.earthengine.google.com/ba0f64848eddfbce06369aa8cdbe21be), which applies these methods to classify vegetation on Santa Cruz Island.
<!-- 
If you have never used GEE before, here is a helpful place to start: [https://jeffhowarth.github.io/eeprimer/start/getGEE/](https://jeffhowarth.github.io/eeprimer/start/getGEE/) -->

### What are the aspects of this tutorial?

1. [**Data inputs and pre-processing**](#upload-or-import-data)
	1. [Satellite imagery](#satellite-imagery) (NAIP, Sentinel, etc.)
	2. [Elevation data](#elevation-data) (Upload or import from GEE)
2. [**Calculate spectral and topographic variables**](#calculate-spectral-and-topographic-variables)
	* [Spectral variables](#spectral-variables)
		* Spectral indices (image bands)
		* Ratio indices
			* Normalized Difference Vegetation Index (NDVI)
			* Burn index ratio
			* Other band ratios
	* [Topographic variables](#topographic-variables)
		* [Canopy height](#canopy-height) (only possible with elevation and surface models)
		* [Heat load index](#heat-load-index)
		* [Topographic position index](#topographic-position-index) (TPI)
		* [Mean TPI](#mean-tpi)
		* [Slope](#slope)
3. [**Set up and apply classification methods**](#set-up-and-apply-classification-methods)
	* [Create image stack with bands and variables](#create-image-stack-with-bands-and-variables)
	* [Create training and validation geometries using stratified sampling](#create-training-and-validation-geometries-using-stratified-sampling)
	* [Filter training and validation data](#filter-training-and-validation-data)
	* [Random Forest classification](#random-forest-classification)
4. [**Evaluate and visualize results**](#evaluate-and-visualize-results)
	* [Confusion matrix](#confusion-matrix)
	* [Variable importance](#variable-importance)
	* [Visualization parameters](#visualization-parameters)
	* [Export data](#export-data)

###Upload or import data

To perform image classification with topographic variables, there are two necessary inputs: [**satellite imagery**] and [**elevation data**]. These inputs will define the spatial resolution and projection of your entire analysis, so it is essential to think carefully about 

####**Satellite imagery** 

Depending upon your study area and end goals, there are several options for satellite imagery housed within GEE. 
National Agricultural Imagery Program (NAIP) and Sentinel data both offer benefits and drawbacks for this type of analysis. NAIP imagery is much higher-resolution (1-meter compared to 10-meter for Sentinel) yet it is possible to calculate seasonal differences with Sentinel, while NAIP does not have enough imagery to allow for this type of analysis.


##### Sentinel
```javascript
//Sentinel imagery has a spatial resolution of 10 meters
var outScale = 10;

//Coordinate system is WGS 84 / UTM zone 11N
var outCRS = 'EPSG:32611'

//Due to the temporal frequency of Sentinel, we can calculate two separate images based on a summer/winter time period.
var startDate1 = '2019-02-15';
var endDate1 = '2019-03-15';
var startDate2 = '2019-08-01';
var endDate2 = '2019-09-30';
var cloudPercentage = 0.2;

//We will utilize two time periods for Sentinel data to be able to compare vegetation change.
var sentinel1 = ct.loadSentinel(extent,startDate1,endDate1,cloudPercentage);
var sentinel2 = ct.loadSentinel(extent,startDate2,endDate2,cloudPercentage);
```

##### NAIP
```javascript
var outScale = 1.5;
var outCRS = 'EPSG:26911'
var year = 2018

//geometry is a hand-drawn polygon in GEE that encompasses your entire study area.
var naip = ct.loadNAIP(geometry);
var naipYears = ct.addYears(naip);
var naipYearsFourBands = ct.filterFourBands(naipYears);
var withNDVI = ct.addNDVI(naipYearsFourBands);
var naipMosaic = ct.mosaicNAIP(withNDVI,year,mask,outCRS,outScale)
```

####**Elevation data** 

There are several options for elevation data in GEE, including Shuttle Radar Topography Mission (SRTM) and USGS National Elevation Dataset (NED). These options offer national (USGS NED) or global (SRTM) elevation rasters, but unfortunately there is no surface model available for this data. 

In this tutorial, we use our own elevation data from LiDAR, which is not included in Earth Engine and offers both elevation and surface models for certain regions. If you have available LiDAR data for your study region, you can upload GeoTIFFs as assets to Earth Engine by clicking on the **Assets** tab, then the **New** button, and upload your GeoTIFF. The following code offers modules for loading a LiDAR DEM and DSM, an associated land mask for our study area, and reducing the resolution of this data to match the imagery (in this case, Sentinel). If we are using NAIP imagery, it would be necessary to reproject in the opposite direction.

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

//Because this case study is for an island, we create a mask to mask out the water. This will make our measurements of certain indicators more accurate due to less noise from water data.
var mask = ct.calculateMask(dem,10,300)
```

If you do not have LiDAR, you can use the USGS NED for study areas within the United States or SRTM data for global case studies. Add these to your script by searching for one of these datasets in the search bar and clicking the **Import** button. It should appear at the top of your script under an **Imports** header and you can change the name to whatever you like. In this script, we will name our elevation data **```dem```**. You will not be able to include the ```dsm``` in your analysis.

```javascript
//SRTM
var dem = ee.Image("USGS/SRTMGL1_003");

//USGS National Elevation Dataset
var dem = ee.Image("USGS/NED");
```

Either way, it will be necesary to mask your ```dem``` and ```dsm```.
```javascript
var maskedDEM = ct.maskDEM(dem, mask);
var maskedDSM = ct.maskDEM(dsm, mask);
```


### Calculate spectral and topographic variables

Once you have successfully loaded your imagery and elevation data, you can begin to calculate spectral and topographic variables that will be used in the classification process. 

#### Topographic variables

Most of our topographic variables utilize only a digital elevation model, yet the canopy height method requires a digital surface model in addition.

##### Canopy Height
Canopy height is calculated by subtracting a digital elevation model from a digital surface model. This calculation is **not possible** if you are using SRTM or USGS NED data, which does not offer a surface model. 
```javascript
var difference = ct.elevationDifference(dem,dsm)
	.reproject(outCRS,null,outScale)
	.resample('bilinear')
```  

##### Slope
The first topographic variable that requires only a DEM is slope, which helps distinguish between different landform types and will be an input to our heat load index.
```javascript
var slopeDegrees = ct.calculateSlopeDegrees(dem);
```

##### Topographic Position Index
Based on Theobald et al (2015), we calculate a multi-scale topographic position index (TPI) that measures relative topographic relief. This is measured by subtracting mean elevation for a neighborhood of cells from the base elevation data. In this case, we calculate TPI with kernel radius of 270m.
```javascript
var tpi_270m = ct.calculateTPI(dem,demMean_270m)
```
[View tpi_270m function description](./modules.md#calculatetpi)

##### Mean TPI
Mean TPI is a helpful factor to be able to distinguish between landform types and requires only a DEM. This workflow is based on Theobald et al's (2015) method for calculating a mean TPI using three resolutions. We calculate the standardized TPI, which is the topographic position index divided by the standard deviation of elevation at the same spatial resolution. These standardized TPIs are then averaged to compute a Mean TPI layer.
```javascript
var demMean_270m = ct.calculateNeighborhoodMean(dem,27)
var demStdDev_270m = ct.calculateNeighborhoodStdDev(dem,27)
var stdTPI_270m = ct.calculateStandardizedTPI(dem,demMean_270m,demStdDev_270m)

var demMean_810m = ct.calculateNeighborhoodMean(dem,81)
var demStdDev_810m = ct.calculateNeighborhoodStdDev(dem,81)
var stdTPI_810m = ct.calculateStandardizedTPI(dem,demMean_810m,demStdDev_810m)

var demMean_2430m = ct.calculateNeighborhoodMean(dem,243)
var demStdDev_2430m = ct.calculateNeighborhoodStdDev(dem,243)
var stdTPI_2430m = ct.calculateStandardizedTPI(maskedDEM,demMean_2430m,demStdDev_2430m)

var meanTPI = ct.calculateMeanTPI(stdTPI_270m,stdTPI_810m,stdTPI_2430m)
```

##### Heat load index
We based our heat load index (HLI) on a workflow developed by SOMEONE (2002) and modified by Theobald et al (2015). This continuous heat-insolation index attempts to measure the amount of heat exposure on a given piece of land based on topography. 
```javascript
var theobaldHLI = ct.calculateTheobaldHLI(dem);
```
[View theobaldHLI function description](./modules.md#theobaldhli)

#### Spectral variables



##### NDVI Difference
##### Forest-cropland
##### NDVI 45
##### Burn ratio
* NAIP - add NDVI, add year, etc
* Sentinel - cloudy


### Set up and apply classification methods

#### Create image stack with bands and variables


#### Create training and validation geometries
The creation of reliable training and validation geometries requires some form of trusted samples. This could include ground-truthed data collected in the field, prior data gathered by other research groups, or hand-selected points/polygons within Earth Engine. 

In general, this step requires reliable geometries (points or polygons) that are associated with the classes that you would like to predict with the Random Forest classification. The first method (reference data) is dependent upon existing data but allows for a much quicker workflow. On the other hand, hand-selecting points or polygons in Earth Engine or QGIS will be much more time intensive yet will potentially be more accurate.

##### Reference data method

For our case study we utilize a vegetation dataset from [2007](https://map.dfg.ca.gov/metadata/ds0563.html) for Santa Cruz Island, the largest of the Channel Islands. This shapefile was downloaded, imported into QGIS, and dissolved on the Veg_Class field to reduce the number of fields for classification. In the end, we outputted three different versions of the data to allow for a comparative analysis:

* Two classes (forest and non-forest)
* Three classes (forest, non-forest, bare)
* Four classes (forest, deciduous, herbaceous, bare)

Once you have output layers with dissolved geometries with your desired number of classes (and integers labeling each class), we can assign an equal number of random points to each class. This step can be completed in either GEE or QGIS. 

In QGIS, it is possible to use the Research Tools tab to *Add Points by Layer* for training and validation data. These point layers can then be uploaded to GEE as an asset. 

In GEE, you can export the classes file as a shapefile, upload the data as an asset, and use the following code to create training and validation data:

```javascript
//Convert uploaded vector feature data to image
var painted = ct.paintImageWithFeatures(twoClasses)

//Create a stratified sample of 'numPointsPerClass' points within a given area
var stratifiedTraining = ct.stratify(painted,numPointsPerClass,outCRS,outScale,validationArea)
var stratifiedValidation = ct.stratify(painted,numPointsPerClass,outCRS,outScale,sampleArea)
```
[View paintImageWithFeatures function description](./modules.md#stratify)
[View stratify function description](./modules.md#stratify)

#### Filter training and validation data
Once you have created the training and validation data, it is possible to filter the training and validation data depending on its reliability. For example, since we are using a 2007 map of vegetation with a higher spatial resolution, we will filter out points that were labeled certain classes and have specific NDVI or elevation difference values.

First, it is necessary to perform a spatial join of the desired bands to the geometries.

```javascript
var ndvi_winter = multiband.select(['0_NDVI'])
var ndvi_summer = multiband.select(['1_NDVI'])

var stratifiedTraining = ct.spatialJoin(ndvi_winter,stratifiedTraining).map(function(f){
	return f.select(['class','first'],['class','ndvi_value'])
})
var stratifiedTraining = ct.spatialJoin(difference,stratifiedTraining).map(function(f){
	return f.select(['class','ndvi_value','first'],['class','ndvi_value','norm_diff'])
})

var stratifiedTraining = ct.filterPointsTwo_Sentinel(stratifiedSample_ndvi_diff,forestNDVIMin_Winter,forestDiffMin,notForestDiffMax,notForest_NDVIMax_Winter)


var stratifiedValidation = ct.spatialJoin(ndvi_winter,stratifiedValidation).map(function(f){
	return f.select(['class','first'],['class','ndvi_value'])
})
var stratifiedValidation = ct.spatialJoin(difference,stratifiedValidation).map(function(f){
	return f.select(['class','ndvi_value','first'],['class','ndvi_value','norm_diff'])
})
var stratifiedValidation = ct.filterPointsTwo_Sentinel(stratifiedValidation_ndvi_diff,forestNDVIMin_Winter,forestDiffMin,notForestDiffMax,notForest_NDVIMax_Winter)
```
#### Random Forest classification
```javascript
// Create training and validation datasets by sampling pixels from the training collection.
var training_data = ct.generateSampleData(multiband_diff,bands,stratifiedTraining,'class');
var validation_data = ct.generateSampleData(multiband_diff,bands,stratifiedValidation,'class');

// Create classifier with numTrees and train on training data.
var RF_trained = ee.Classifier.smileRandomForest(numTrees).train(training_data,'class',bands)

// Classify validation data with trained classifier.
var RF_validation = validation_data.classify(RF_trained);
```
### Evaluate and visualize results
#### Compute confusion matrix
```javascript
//Produce an error matrix and output accuracy values for Random Forest
var RF_matrix = RF_validation.errorMatrix('class', 'classification');
var RF_dict = ct.createRFDict(RF_matrix,numClasses)
print("Random Forest accuracy information",RF_dict)
```

#### Variable importance
```javascript
var explained = RF_trained.explain()
print('Explanation of trained RF model',explained)
```
#### Visualization parameters
#### Export data?
