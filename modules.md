## Modules for image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5</br>
[**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

### Introduction
This overview of the modules will be organized in the same way as the [module script](https://code.earthengine.google.com/44c19d132dd87e5d8e0fc904d1af8f09), reflecting the major tasks that are involved in performing this type of analysis:

1. [IMPORT ASSETS](#import-assets)
2. [LOAD and FILTER IMAGERY](#load-and-filter-imagery)
	* [loadNAIP](#loadnaip)
	* [loadSentinel](#loadsentinel)
3. [PRE-PROCESS IMAGERY](#pre-process-imagery)
	* [addYears](#addyears)
	* [filterFourBands](#filterfourbands)
	* [addNDVI](#addndvi)
	* [addNDVISentinel](#addndvisentinel)
	* [addPercentDiffSentinel](#addpercentdiffsentinel)
	* [addBurnSentinel](#addburnsentinel)
4. [DATA PROCESSING](#data-processing)
5. [TOPOGRAPHIC VARIABLES](#topographic-variables)
6. [MULTIBAND IMAGE ANALYSIS METHODS](#multiband-image-analysis-methods)
7. [VISUALIZATION](#visualization)


### IMPORT ASSETS
<!-- ```javascript
``` -->
### LOAD and FILTER IMAGERY
##### loadNAIP
```javascript
//  Load NAIP from GEE library and filter to study area  
exports.loadNAIP =(extent) {
  return ee.ImageCollection("USDA/NAIP/DOQQ")
            .filterBounds(extent)
}
```

##### loadSentinel
```javascript
//Load Sentinel from GEE library and filter to study area, time period, and cloudy percentage
exports.loadSentinel =(extent,startDate,endDate,cloudPercentage){
  return ee.ImageCollection('COPERNICUS/S2_SR')
            .filterBounds(extent)
            .filterDate(startDate, endDate)
            .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',cloudPercentage));
}
```
### PRE-PROCESS IMAGERY

##### addYears
```javascript
//Add year as an attribute to a collection of NAIP images
exports.addYears =(naipImages){
  return naipImages.map(image){
    var imageYear = image.date().get('year')
    var bandNames = image.bandNames().length()
    return image.set({year:imageYear, bands:bandNames})
  })
}
```
##### filterFourBands
```javascript
//Filter NAIP images based on whether there are at least four bands
exports.filterFourBands =(naipImagesWithYears){
  return naipImagesWithYears.filter(ee.Filter.gt('bands', 3));
}
```

##### addPercentDiffSentinel
```javascript
exports.addPercentDiffSentinel =(image,bandName){
  var added = (image.select('1_'+bandName).subtract(image.select('0_'+bandName))).divide(image.select('1_'+bandName)).rename(bandName+'_diff');
  return image.addBands(added);
}
```

##### addBandRatioSentinel
```javascript
exports.addBandRatioSentinel =(image,bandName){
  var added = image.normalizedDifference(['0_'+bandName, '1_'+bandName]).rename(bandName+'_diff');
  return image.addBands(added);
}
```

##### addNDVI
```javascript
// Add NDVI attribute to all NAIP images in a collection
exports.addNDVI =(images){
  return images.map(image){
    var ndvi = image.normalizedDifference(['N', 'R']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
```
```javascript
// Add NDVI attribute to all Sentinel images in a collection
exports.addNDVISentinel =(images){
  return images.map(image){
    var ndvi = image.normalizedDifference(['B8', 'B4']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
```
```javascript
// Add burn ratio attribute to all Sentinel images in a collection
exports.addBurnSentinel =(images){
  return images.map(image){
    var burn = image.normalizedDifference(['B8', 'B12']).rename('burn');
  return image.addBands(burn);
  });
}
```
```javascript
//Add band ratio for Sentinel imagery from:
//tm7-tm2 from http://web.pdx.edu/~nauna/resources/10_BandCombinations.htm
exports.addForestCroplandSentinel =(images){
  return images.map(image){
    var forestCropland = image.normalizedDifference(['B12', 'B3']).rename('forest');
  return image.addBands(forestCropland);
  });
}
```
```javascript
//Add NDVI 45 attribute to all Sentinel images in a collection
exports.addNDVI45 =(images){
  return images.map(image){
    var ndvi45 = image.normalizedDifference(['B5', 'B4']).rename('NDVI45');
  return image.addBands(ndvi45);
  });
}
```
```javascript
//Mosaic NAIP images from a collection and reproject
exports.mosaicNAIP =(images,year,mask,outCRS,outScale){
  return images.filter(ee.Filter.calendarRange(year, year, 'year'))
    .mosaic()
    .set({year:year})
    .mask(mask)
    .reproject(outCRS,null,outScale)
    .resample('bilinear')
    .mask(mask)
}
```
```javascript
//Mosaic Sentinel images from a collection using a quality attribute and reproject
exports.mosaicSentinel =(images,mask,outCRS,outScale){
  return images
    .qualityMosaic('QA10')
    .mask(mask)
    .reproject(outCRS,null,outScale)
    .resample('bilinear')
    .mask(mask)
}
```
### DATA PROCESSING
<!-- ```javascript
``` -->
### TOPOGRAPHIC VARIABLES
<!-- ```javascript
``` -->
### MULTIBAND IMAGE ANALYSIS METHODS
<!-- ```javascript
``` -->
### VISUALIZATION
<!-- ```javascript
``` -->
