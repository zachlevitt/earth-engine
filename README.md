![](header.png "Image classification in Earth Engine")

## Image classification in Google Earth Engine with topographic and spectral variables
**Zach Levitt**, '20.5 and **Jeff Howarth**, Associate Professor of Geography</br>
Middlebury College, Vermont, USA

This tutorial will introduce a workflow for utilizing topographic, spectral and spatial variables to classifying remotely-sensed imagery using machine learning approaches in Google Earth Engine (GEE). Using a combination of pre-loaded GEE satellite imagery and analysis methods, as well as topographic variables derived from uploaded LiDAR data, we will outline a novel image classification methodology. We will apply these tools to classify vegetation on the Channel Islands of California, though the tools are applicable to a wide range of use cases. We will utilize both supervised and unsupervised classification methods to show various methods for improving the accuracy and efficiency of the workflow.

There will be code snippets throughout the tutorial and you can view the completed script [here](https://code.earthengine.google.com/ba0f64848eddfbce06369aa8cdbe21be).

<!-- ### Background

There are two primary motiviations for this work:

1. Apply machine learning approaches to identify vegetation classes using topographic, spectral and spatial variables.
2. Update vegetation data for the Channel Islands to aid conservation and environmental projects (Last updated in [2007](http://iws.org/CISProceedings/7th_CIS_Proceedings/Cohen_et_al.pdf) for [Santa Cruz Island](https://map.dfg.ca.gov/metadata/ds0563.html), the largest of the Channel Islands).
 -->

### Introduction

* We will use modules
* We will reproject to ....
* Define variables like number of classes, bands, scale, etc
* projection if naip, or sentinel 
* cloudy
* If you have never used GEE before, here is a helpful place to start [https://jeffhowarth.github.io/eeprimer/start/getGEE/](https://jeffhowarth.github.io/eeprimer/start/getGEE/)
* Our code will be broken down into two main files - modules and function calls that bring it all together

### Data inputs

There are several key data inputs for this analysis, including:

* Satellite imagery (NAIP, Sentinel)
* Elevation data (LiDAR)
* Land mask

### Data pre-processing 

* NAIP - add NDVI, add year, etc
* Sentinel - cloudy
* Elevation data - we used ArcMap to gather and clean LiDAR data, if this is not available you could use SRTM or DEM USGS data. But we wanted 1.5 m data
* 

### Modules

* chisTools.js - call functions

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





