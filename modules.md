## Modules for image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5</br>
[**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

### Introduction
These module functions are provided here as a reference for the [main tutorial](./README.md), which outlines a workflow for performing machine learning image classification with topographic and spectral variables. This overview is organized in the same way as the [module script](https://code.earthengine.google.com/44c19d132dd87e5d8e0fc904d1af8f09), reflecting the major tasks that are involved in performing this type of analysis. Each function and section is linked below.

1. [**IMPORT ASSETS**](#import-assets)
2. [**LOAD and FILTER IMAGERY**](#load-and-filter-imagery)
	* [loadNAIP](#loadnaip)
	* [loadSentinel](#loadsentinel)
3. [**PRE-PROCESS IMAGERY**](#pre-process-imagery)
	* [addYears](#addyears)
	* [filterFourBands](#filterfourbands)
	* [addNDVI](#addndvi)
	* [addNDVISentinel](#addndvisentinel)
	* [addPercentDiffSentinel](#addpercentdiffsentinel)
	* [addBurnSentinel](#addburnsentinel)
	* [addForestCroplandSentinel](#addforestcroplandsentinel)
	* [addNDVI45](#addndvi45)
	* [mosaicNAIP](#mosaicnaip)
	* [mosaicSentinel](#mosaicsentinel)
4. [**DATA PROCESSING**](#data-processing)
5. [**TOPOGRAPHIC VARIABLES**](#topographic-variables)
	* [elevationDifference](#elevationdifference)
	* [calculateSlopeDegrees](#calculateslopedegrees)
	* [calculateTheobaldHLI](#calculatetheobaldhli)
	* [calculateNeighborhoodMean](#calculateneighborhoodmean)
	* [calculateNeighborhoodStdDev](#calculateneighborhoodstddev)
	* [calculateStandardizedTPI](#calculatestandardizedtpi)
	* [calculateMeanTPI](#calculatemeantpi)
	* [calculateTPI](#calculatetpi)
6. [**MULTIBAND IMAGE ANALYSIS**](#multiband-image-analysis-methods)
	* [toBandedImage](#tobandedimage)
	* [generateSampleData](#generatesampledata)
	* [paintImageWithFeatures](#paintimagewithfeatures)
	* [stratify](#stratify)
	* [spatialJoin](#spatialjoin)
	* [filterPointsFour](#filterpointsfour)
	* [filterPointsThree](#filterpointsthree)
	* [filterPointsTwo](#filterpointstwo)
7. [VISUALIZATION](#visualization)

[**Return to main tutorial**](./README.md)


### IMPORT ASSETS
<!-- ```javascript
``` -->
### LOAD and FILTER IMAGERY
##### loadNAIP
```javascript
//  Load NAIP from GEE library and filter to study area  
exports.loadNAIP = function(extent) {
  return ee.ImageCollection("USDA/NAIP/DOQQ")
            .filterBounds(extent)
}
```

##### loadSentinel
```javascript
//Load Sentinel from GEE library and filter to study area, time period, and cloudy percentage
exports.loadSentinel = function(extent,startDate,endDate,cloudPercentage){
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
exports.addYears = function(naipImages){
  return naipImages.map(function(image){
    var imageYear = image.date().get('year')
    var bandNames = image.bandNames().length()
    return image.set({year:imageYear, bands:bandNames})
  })
}
```
##### filterFourBands
```javascript
//Filter NAIP images based on whether there are at least four bands
exports.filterFourBands = function(naipImagesWithYears){
  return naipImagesWithYears.filter(ee.Filter.gt('bands', 3));
}
```

##### addPercentDiffSentinel
```javascript
exports.addPercentDiffSentinel = function(image,bandName){
  var added = (image.select('1_'+bandName).subtract(image.select('0_'+bandName))).divide(image.select('1_'+bandName)).rename(bandName+'_diff');
  return image.addBands(added);
}
```

##### addBandRatioSentinel
```javascript
exports.addBandRatioSentinel = function(image,bandName){
  var added = image.normalizedDifference(['0_'+bandName, '1_'+bandName]).rename(bandName+'_diff');
  return image.addBands(added);
}
```

##### addNDVI
```javascript
// Add NDVI attribute to all NAIP images in a collection
exports.addNDVI = function(images){
  return images.map(function(image){
    var ndvi = image.normalizedDifference(['N', 'R']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
```

###### addNDVISentinel
```javascript
// Add NDVI attribute to all Sentinel images in a collection
exports.addNDVISentinel = function(images){
  return images.map(function(image){
    var ndvi = image.normalizedDifference(['B8', 'B4']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
```
###### addBurnSentinel
```javascript
// Add burn ratio attribute to all Sentinel images in a collection
exports.addBurnSentinel = function(images){
  return images.map(function(image){
    var burn = image.normalizedDifference(['B8', 'B12']).rename('burn');
  return image.addBands(burn);
  });
}
```
##### addForestCroplandSentinel
```javascript
//Add band ratio for Sentinel imagery from:
//tm7-tm2 from http://web.pdx.edu/~nauna/resources/10_BandCombinations.htm
exports.addForestCroplandSentinel = function(images){
  return images.map(function(image){
    var forestCropland = image.normalizedDifference(['B12', 'B3']).rename('forest');
  return image.addBands(forestCropland);
  });
}
```

##### addNDVI45
```javascript
//Add NDVI 45 attribute to all Sentinel images in a collection
exports.addNDVI45 = function(images){
  return images.map(function(image){
    var ndvi45 = image.normalizedDifference(['B5', 'B4']).rename('NDVI45');
  return image.addBands(ndvi45);
  });
}
```

##### mosaicNAIP
```javascript
//Mosaic NAIP images from a collection and reproject
exports.mosaicNAIP = function(images,year,mask,outCRS,outScale){
  return images.filter(ee.Filter.calendarRange(year, year, 'year'))
    .mosaic()
    .set({year:year})
    .mask(mask)
    .reproject(outCRS,null,outScale)
    .resample('bilinear')
    .mask(mask)
}
```
##### mosaicSentinel
```javascript
//Mosaic Sentinel images from a collection using a quality attribute and reproject
exports.mosaicSentinel = function(images,mask,outCRS,outScale){
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
##### elevationDifference
```javascript
//Compute Canopy Height Model
exports.elevationDifference = function(dem,dsm){
  return dsm.subtract(dem).rename('diff')
}
```

##### calculateSlopeDegrees
```javascript
//Calculate slope in degrees from elevation data
exports.calculateSlopeDegrees = function(dem,mask){
  return ee.Terrain.slope(dem).mask(mask)
}
```

##### calculateTheobaldHLI
```javascript
//Calculate heat load index, based on Theobald et al (2015)
exports.calculateTheobaldHLI = function(dem,mask) {
  var aspectRadians = ee.Terrain.aspect(dem).mask(mask).multiply(ee.Number(0.01745329252))
  var slopeRadians = ee.Terrain.slope(dem).mask(mask).multiply(ee.Number(0.01745329252))
  var foldedAspect = aspectRadians.subtract(ee.Number(4.3196899)).abs().multiply(ee.Number(-1)).add(ee.Number(3.141593)).abs()
  var theobaldHLI = (slopeRadians.cos().multiply(ee.Number(1.582)).multiply(ee.Number(0.82886954044))).subtract(foldedAspect.cos().multiply(ee.Number(1.5)).multiply(slopeRadians.sin().multiply(ee.Number(0.55944194062)))).subtract(slopeRadians.sin().multiply(ee.Number(0.262)).multiply(ee.Number(0.55944194062))).add(foldedAspect.sin().multiply(ee.Number(0.607)).multiply(slopeRadians.sin())).add(ee.Number(-1.467));
  return theobaldHLI.exp().rename('hli')
};
```

##### calculateNeighborhoodMean
```javascript
//  Calculate a neighborhood mean using a pre-defined kernel radius
exports.calculateNeighborhoodMean = function(image, kernelRadius) {
  return image.reduceNeighborhood({
    reducer: ee.Reducer.mean(),
    kernel: ee.Kernel.square(kernelRadius,'pixels',false),
    optimization: 'boxcar',
  });
}
```

##### calculateNeighborhoodStdDev
```javascript
//Calculate a neighborhood standard deviation using a pre-defined kernel radius    
exports.calculateNeighborhoodStdDev = function(image, kernelRadius) {
  return image.reduceNeighborhood({
    reducer: ee.Reducer.stdDev(),
    kernel: ee.Kernel.square(kernelRadius,'pixels',false),
    optimization: 'window',
  });
}
```

##### calculateTPI
```javascript
//Calculate topographic position index from a pre-defined mean image
exports.calculateTPI = function(image, meanImage) {
  return image.subtract(meanImage).rename('tpi')
}
```

##### calculateMeanTPI
```javascript
//Calculate mean topographic position index, based on Theobald et al (2015)
exports.calculateMeanTPI = function(image1,image2,image3){
  return (image1.add(image2).add(image3)).divide(ee.Number(3)).rename('meanTPI')
}
```

### MULTIBAND IMAGE ANALYSIS METHODS

##### toBandedImage
```javascript
//Add aligned rasters to a single image as bands
exports.toBandedImage = function(imageCollection){
  return imageCollection.toBands()
}
```

##### generateSampleData
```javascript
//Add aligned rasters to a single image as bands
exports.generateSampleData = function(image,bands,collection,property){
  return image.select(bands).sampleRegions({
    collection: collection,
    properties: [property]
  })
}
```

##### paintImageWithFeatures
```javascript
exports.paintImageWithFeatures = function(features){
  return ee.Image().byte().paint({
    featureCollection: features,
    color: 'Class',
  }).rename('class')
}
```

##### stratify
```javascript
exports.stratify = function(image,numPointsPerClass,outCRS,outScale,geometry){
  return image.addBands(ee.Image.pixelLonLat())
    .stratifiedSample({
      numPoints: numPointsPerClass,
      classBand: 'class',
      projection:outCRS,
      scale: outScale,
      region: geometry,
    }).map(function(f) {
      return f.setGeometry(ee.Geometry.Point([f.get('longitude'), f.get('latitude')]))
  })
}
```

##### spatialJoin
```javascript
//Spatial join of raster data to vector points
exports.spatialJoin = function(image,points){
  return image.reduceRegions(points,ee.Reducer.first())
}
```

##### filterPointsFour
```javascript
exports.filterPointsFour = function(features,forestNDVI,forestDiff,bareNDVI,bareDiff,herbDiff,herbNDVI){
  var filter1 = features.filter(ee.Filter.and(ee.Filter.eq('class', 4),ee.Filter.gt('ndvi_value', bareNDVI),ee.Filter.gt('norm_diff',bareDiff)).not());
  var filter2 = filter1.filter(ee.Filter.and(ee.Filter.eq('class', 2),ee.Filter.gt('ndvi_value', herbNDVI)).not());
  var filter3 = filter2.filter(ee.Filter.and(ee.Filter.eq('class', 1),ee.Filter.lt('ndvi_value', forestNDVI),ee.Filter.lt('norm_diff',forestDiff)).not());
  var filter4 = filter3.filter(ee.Filter.and(ee.Filter.eq('class', 3),ee.Filter.gt('norm_diff',herbDiff),ee.Filter.lt('ndvi_value', herbNDVI)).not())
  return filter4;
}
```

##### filterPointsThree
```javascript  
exports.filterPointsThree = function(features,forestNDVI,forestDiff,bareNDVI,bareDiff,herbDiff,herbNDVI){
  var filter1 = features.filter(ee.Filter.and(ee.Filter.eq('class', 2),ee.Filter.gt('ndvi_value', bareNDVI),ee.Filter.gt('norm_diff',bareDiff)).not());
  var filter2 = filter1.filter(ee.Filter.and(ee.Filter.eq('class', 0),ee.Filter.lt('ndvi_value', forestNDVI),ee.Filter.lt('norm_diff',forestDiff)).not());
  var filter3 = filter2.filter(ee.Filter.and(ee.Filter.eq('class', 1),ee.Filter.lt('norm_diff',herbDiff),ee.Filter.lt('ndvi_value', herbNDVI)).not())
  return filter3;
}
```

##### filterPointsThree
```javascript   
exports.filterPointsThree = function(features,forestNDVI,forestDiff,bareNDVI,bareDiff,herbDiff,herbNDVI){
  var filter1 = features.filter(ee.Filter.and(ee.Filter.eq('class', 2),ee.Filter.gt('ndvi_value', bareNDVI),ee.Filter.gt('norm_diff',bareDiff)).not());
  var filter2 = filter1.filter(ee.Filter.and(ee.Filter.eq('class', 0),ee.Filter.lt('ndvi_value', forestNDVI),ee.Filter.lt('norm_diff',forestDiff)).not());
  var filter3 = filter2.filter(ee.Filter.and(ee.Filter.eq('class', 1),ee.Filter.lt('norm_diff',herbDiff),ee.Filter.lt('ndvi_value', herbNDVI)).not())
  return filter3;
}
```

### VISUALIZATION
<!-- ```javascript
``` -->
