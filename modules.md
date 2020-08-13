# Modules for image classification in Google Earth Engine with topographic and spectral variables
[**Zach Levitt**](https://zachlevitt.github.io), '20.5</br>
[**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

These module functions are provided here as a reference for the [main tutorial](./README.md), which outlines a workflow for performing machine learning image classification with topographic and spectral variables. This overview is organized in the same way as the [module script](https://code.earthengine.google.com/44c19d132dd87e5d8e0fc904d1af8f09), reflecting the major tasks that are involved in performing this type of analysis. Each function and section is linked below.

1. [**IMPORT ASSETS**](#import-assets)
2. [**LOAD and FILTER IMAGERY**](#load-and-filter-imagery)
	* [loadImageAsset](#loadImageAsset)
	* [loadSentinel](#loadsentinel)
3. [**PRE-PROCESS IMAGERY**](#pre-process-imagery)
	* [addNDVISentinel](#addndvisentinel)
	* [addPercentDiffSentinel](#addpercentdiffsentinel)
	* [addBurnSentinel](#addburnsentinel)
	* [addForestCroplandSentinel](#addforestcroplandsentinel)
	* [addNDVI45](#addndvi45)
	* [mosaicNAIP](#mosaicnaip)
	* [mosaicSentinel](#mosaicsentinel)
4. [**DEM PROCESSING**](#dem-processing)
	* [calculateMask](#calculatemask)
	* [processDEM](#processDEM)
	* [resampleDEM](#resampleDEM)
5. [**TOPOGRAPHIC VARIABLES**](#topographic-variables)
	* [elevationDifference](#elevationdifference)
	* [calculateSlopeDegrees](#calculateslopedegrees)
	* [calculateTheobaldHLI](#calculatetheobaldhli)
	* [calculateNeighborhoodMean](#calculateneighborhoodmean)
	* [calculateNeighborhoodStdDev](#calculateneighborhoodstddev)
	* [calculateStandardizedTPI](#calculatestandardizedtpi)
	* [calculateMeanTPI](#calculatemeantpi)
	* [calculateTPI](#calculatetpi)
	* [calculateLandforms](#calculatelandforms)
	* [remapLandforms](#remaplandforms)
6. [**MULTIBAND IMAGE ANALYSIS**](#multiband-image-analysis-methods)
	* [toBandedImage](#tobandedimage)
	* [generateSampleData](#generatesampledata)
	* [paintImageWithFeatures](#paintimagewithfeatures)
	* [createStratifiedPoints](#createStratifiedPoints)
	* [spatialJoin](#spatialjoin)
	* [filterPoints](#filterPoints)
	* [createRFDict](#createRFDict)
7. [VISUALIZATION](#visualization)

[**Return to main tutorial**](./README.md)


## IMPORT ASSETS
<!-- ```javascript
``` -->
## LOAD and FILTER IMAGERY
### loadImageAsset
Input: path (string)
Output: ee.Image
```javascript
exports.loadImageAsset = function(path){
    return ee.Image(path)
  }
```

### loadNAIP
Input: extent (ee.Feature or ee.FeatureCollection), studyYear (Number)
Output: ee.ImageCollection
```javascript
//Loads NAIP imagery for a given extent and study year.
//Adds an NDVI band and filters for imagery with >3 bands
exports.loadNAIP = function(extent,studyYear){
    return ee.ImageCollection("USDA/NAIP/DOQQ")
      .filterBounds(extent)
      .map(function(image){
        var imageYear = image.date().get('year')
        var bandNames = image.bandNames().length()
        return image.set({year:imageYear, bands:bandNames})
      })
      .filter(ee.Filter.gt('bands', 3))
      .map(function(image){
        var ndvi = image.normalizedDifference(['N', 'R']).rename('NDVI');
        return image.addBands(ndvi);
      });
}
```

### loadSentinel
```javascript
//Loads Sentinel 2-A imagery for a given extent, start and end date, and max cloud percentage.
//Filters imagery for cloudy pixels
exports.loadSentinel = function(extent,startDate,endDate,cloudPercentage){
    return ee.ImageCollection("COPERNICUS/S2_SR")
      .filterBounds(extent)
      .filterDate(startDate, endDate)
      .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',cloudPercentage))
      .map(function(image){
        var qa = image.select('QA60');
            
        // Bits 10 and 11 are clouds and cirrus, respectively.
        var cloudBitMask = 1 << 10;
        var cirrusBitMask = 1 << 11;
            
        // Both flags should be set to zero, indicating clear conditions.
        var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
          .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
            
        return image.updateMask(mask).divide(10000);
      });
}
```
## PRE-PROCESS IMAGERY

<!-- ### addYears
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
### filterFourBands
```javascript
//Filter NAIP images based on whether there are at least four bands
exports.filterFourBands = function(naipImagesWithYears){
  return naipImagesWithYears.filter(ee.Filter.gt('bands', 3));
}
``` -->

### addPercentDiffSentinel
```javascript
exports.addPercentDiffSentinel = function(image,bandName){
  var added = (image.select('1_'+bandName).subtract(image.select('0_'+bandName))).divide(image.select('1_'+bandName)).rename(bandName+'_diff');
  return image.addBands(added);
}
```

### addBandRatioSentinel
```javascript
exports.addBandRatioSentinel = function(image,bandName){
  var added = image.normalizedDifference(['0_'+bandName, '1_'+bandName]).rename(bandName+'_diff');
  return image.addBands(added);
}
```

### addNDVI
```javascript
// Add NDVI attribute to all NAIP images in a collection
exports.addNDVI = function(images){
  return images.map(function(image){
    var ndvi = image.normalizedDifference(['N', 'R']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
```

<!-- ### addNDVISentinel
```javascript
// Add NDVI attribute to all Sentinel images in a collection
exports.addNDVISentinel = function(images){
  return images.map(function(image){
    var ndvi = image.normalizedDifference(['B8', 'B4']).rename('NDVI');
  return image.addBands(ndvi);
  });
}
``` -->
<!-- ### addBurnSentinel
```javascript
// Add burn ratio attribute to all Sentinel images in a collection
exports.addBurnSentinel = function(images){
  return images.map(function(image){
    var burn = image.normalizedDifference(['B8', 'B12']).rename('burn');
  return image.addBands(burn);
  });
}
``` -->
<!-- ### addForestCroplandSentinel
```javascript
//Add band ratio for Sentinel imagery from:
//tm7-tm2 from http://web.pdx.edu/~nauna/resources/10_BandCombinations.htm
exports.addForestCroplandSentinel = function(images){
  return images.map(function(image){
    var forestCropland = image.normalizedDifference(['B12', 'B3']).rename('forest');
  return image.addBands(forestCropland);
  });
}
``` -->

<!-- ### addNDVI45
```javascript
//Add NDVI 45 attribute to all Sentinel images in a collection
exports.addNDVI45 = function(images){
  return images.map(function(image){
    var ndvi45 = image.normalizedDifference(['B5', 'B4']).rename('NDVI45');
  return image.addBands(ndvi45);
  });
}
``` -->

### mosaicNAIP
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
### mosaicSentinel
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
## DEM PROCESSING
### calculateMask
```javascript
//Calculates a mask for a digital elevation model of an islannd
//based on a provided minimum elevation and buffer distance.
exports.calculateMask = function(dem,minElevation,bufferDistance){
  return ee.Image(1)
    .cumulativeCost({
      source: dem.gt(minElevation).selfMask(), 
      maxDistance: bufferDistance,
    }).lt(bufferDistance)
}
```

### processElevationData
```javascript
//Masks image and reprojects/resamples if crs and scale are provided
exports.processElevationData = function(image,mask,crs,scale) {
  if (crs === null){
    return image.mask(mask);
  }
  else {
    return image
    .reduceResolution({
      reducer: ee.Reducer.mean(),
      maxPixels: 1024
    })
    // Request the data at the scale and projection of the MODIS image.
    .reproject({
      crs: crs,
      scale: scale
    })
    .mask(mask)
  }
};
```
### resampleDEM 
```javascript
//Reproject DEM to desired CRS and scale, resample with bilinear method
exports.resampleDEM = function(dem,outCRS,outScale){
  return dem.reproject(outCRS,null,outScale).resample('bilinear')
}
``` 

## TOPOGRAPHIC VARIABLES
### elevationDifference
```javascript
//Compute Canopy Height Model
exports.elevationDifference = function(dem,dsm){
  return dsm.subtract(dem).rename('diff')
}
```

### calculateSlopeDegrees
```javascript
//Calculate slope in degrees from elevation data
exports.calculateSlopeDegrees = function(dem,mask){
  return ee.Terrain.slope(dem).mask(mask)
}
```

### calculateTheobaldHLI
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

### calculateNeighborhoodMean
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

### calculateNeighborhoodStdDev
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

### calculateTPI
```javascript
//Calculate topographic position index from a pre-defined mean image
exports.calculateTPI = function(image, meanImage) {
  return image.subtract(meanImage).rename('tpi')
}
```

### calculateMeanTPI
```javascript
//Calculate mean topographic position index, based on Theobald et al (2015)
exports.calculateMeanTPI = function(image1,image2,image3){
  return (image1.add(image2).add(image3)).divide(ee.Number(3)).rename('meanTPI')
}
```

### calculateLandforms
```javascript
exports.calculateLandforms = function(dem,slopeDegrees,theobaldHLI,meanTPI,tpi_270m){
  var slopeReclass = ee.Image(1)
      .where(slopeDegrees.gt(50), 5000)
      .where(slopeDegrees.gt(2).and(slopeDegrees.lte(50)), 1000)
      .where(slopeDegrees.lte(2), 2000)
      .selfMask();
      
  var theobaldHLIReclass = ee.Image(1)
        .where(theobaldHLI.lte(0.448), 100)
        .where(theobaldHLI.gt(0.448).and(theobaldHLI.lte(0.767)), 200)
        .where(theobaldHLI.gt(0.767), 300)
        .selfMask();
        
  var meanTPIReclass = ee.Image(1)
        .where(meanTPI.lte(-1.2), 10)
        .where(meanTPI.gt(-1.2).and(meanTPI.lte(-0.75)), 20)
        .where(meanTPI.gt(-0.75).and(meanTPI.lte(0)), 30)
        .where(meanTPI.gt(0),40)
        .selfMask();
  
  var tpi_270mReclass = ee.Image(1)
        .where(tpi_270m.lte(-5), 1)
        .where(tpi_270m.gt(-5).and(tpi_270m.lte(0)), 2)
        .where(tpi_270m.gt(0).and(tpi_270m.lte(30)), 3)
        .where(tpi_270m.gt(30).and(tpi_270m.lte(300)), 4)
        .where(tpi_270m.gt(300),5)
        .selfMask();

var reclassCombined = slopeReclass.add(theobaldHLIReclass).add(meanTPIReclass).add(tpi_270mReclass).updateMask(dem);

return reclassCombined
      .where(reclassCombined.eq(2344)
        .or(reclassCombined.eq(1344)),11)
      .where(reclassCombined.eq(1244),12)
      .where(reclassCombined.eq(1144),13)
      .where(reclassCombined.eq(1145)
        .or(reclassCombined.eq(1245))
        .or(reclassCombined.eq(1345))
        .or(reclassCombined.eq(2345)),14)
      .where(reclassCombined.gte(5000).and(reclassCombined.lte(6000)),15)
      .where(reclassCombined.eq(1341)
        .or(reclassCombined.eq(1342))
        .or(reclassCombined.eq(1343)),21)
      .where(reclassCombined.eq(1241)
        .or(reclassCombined.eq(1242))
        .or(reclassCombined.eq(1243)),22)
      .where(reclassCombined.eq(1141)
        .or(reclassCombined.eq(1142))
        .or(reclassCombined.eq(1143)),23)
      .where(reclassCombined.eq(2341)
        .or(reclassCombined.eq(2342))
        .or(reclassCombined.eq(2343)),24)
      .where(reclassCombined.eq(1331)
        .or(reclassCombined.eq(1332))
        .or(reclassCombined.eq(1333))
        .or(reclassCombined.eq(1334))
        .or(reclassCombined.eq(1335)),31)
      .where(reclassCombined.eq(1231)
        .or(reclassCombined.eq(1232))
        .or(reclassCombined.eq(1233))
        .or(reclassCombined.eq(1234))
        .or(reclassCombined.eq(1235)),32)
      .where(reclassCombined.eq(1131)
        .or(reclassCombined.eq(1132))
        .or(reclassCombined.eq(1133))
        .or(reclassCombined.eq(1134))
        .or(reclassCombined.eq(1135)),33)
      .where(reclassCombined.eq(2332)
        .or(reclassCombined.eq(2333))
        .or(reclassCombined.eq(2334))
        .or(reclassCombined.eq(2335))
        .or(reclassCombined.eq(2331)),34)
      .where(reclassCombined.eq(1112)
        .or(reclassCombined.eq(1113))
        .or(reclassCombined.eq(1121))
        .or(reclassCombined.eq(1122))
        .or(reclassCombined.eq(1123))
        .or(reclassCombined.eq(1124))
        .or(reclassCombined.eq(1212))
        .or(reclassCombined.eq(1213))
        .or(reclassCombined.eq(1221))
        .or(reclassCombined.eq(1222))
        .or(reclassCombined.eq(1223))
        .or(reclassCombined.eq(1224))
        .or(reclassCombined.eq(1312))
        .or(reclassCombined.eq(1313))
        .or(reclassCombined.eq(1321))
        .or(reclassCombined.eq(1322))
        .or(reclassCombined.eq(1323))
        .or(reclassCombined.eq(1324))
        .or(reclassCombined.eq(2312))
        .or(reclassCombined.eq(2313))
        .or(reclassCombined.eq(2321))
        .or(reclassCombined.eq(2322))
        .or(reclassCombined.eq(2323))
        .or(reclassCombined.eq(2324)),41)
      .where(reclassCombined.eq(1211)
          .or(reclassCombined.eq(1311))
          .or(reclassCombined.eq(2311))
          .or(reclassCombined.eq(1111)),42)
      .updateMask(reclassCombined.select('constant').gt(999))
}
```

### remapLandforms
```javascript
//Reclassify landforms layer to fit with visualization parameters.
exports.remapLandforms = function(landforms){
  return landforms.remap([11,12,13,14,15,21,22,23,24,31,32,33,34,41,42],[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14])
}
```

## MULTIBAND IMAGE ANALYSIS METHODS

### toBandedImage
```javascript
//Add aligned rasters to a single image as bands
exports.toBandedImage = function(imageCollection){
  return imageCollection.toBands()
}
```

### generateSampleData
```javascript
//Add aligned rasters to a single image as bands
exports.generateSampleData = function(image,bands,collection,property){
  return image.select(bands).sampleRegions({
    collection: collection,
    properties: [property]
  })
}
```

### paintImageWithFeatures
```javascript
exports.paintImageWithFeatures = function(features){
  return ee.Image().byte().paint({
    featureCollection: features,
    color: 'Class',
  }).rename('class')
}
```

### createStratifiedPoints
```javascript
exports.createStratifiedPoints = function(image,numPointsPerClass,outCRS,outScale,geometry){
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

### spatialJoin
```javascript
//Spatial join of raster data to vector points
exports.spatialJoin = function(image,points){
  return image.reduceRegions(points,ee.Reducer.first())
}
```

<!-- ### filterPointsFour
```javascript
exports.filterPointsFour = function(features,forestNDVI,forestDiff,bareNDVI,bareDiff,herbDiff,herbNDVI){
  var filter1 = features.filter(ee.Filter.and(ee.Filter.eq('class', 4),ee.Filter.gt('ndvi_value', bareNDVI),ee.Filter.gt('norm_diff',bareDiff)).not());
  var filter2 = filter1.filter(ee.Filter.and(ee.Filter.eq('class', 2),ee.Filter.gt('ndvi_value', herbNDVI)).not());
  var filter3 = filter2.filter(ee.Filter.and(ee.Filter.eq('class', 1),ee.Filter.lt('ndvi_value', forestNDVI),ee.Filter.lt('norm_diff',forestDiff)).not());
  var filter4 = filter3.filter(ee.Filter.and(ee.Filter.eq('class', 3),ee.Filter.gt('norm_diff',herbDiff),ee.Filter.lt('ndvi_value', herbNDVI)).not())
  return filter4;
}
``` -->
<!-- 
### filterPointsThree
```javascript  
exports.filterPointsThree = function(features,forestNDVI,forestDiff,bareNDVI,bareDiff,herbDiff,herbNDVI){
  var filter1 = features.filter(ee.Filter.and(ee.Filter.eq('class', 2),ee.Filter.gt('ndvi_value', bareNDVI),ee.Filter.gt('norm_diff',bareDiff)).not());
  var filter2 = filter1.filter(ee.Filter.and(ee.Filter.eq('class', 0),ee.Filter.lt('ndvi_value', forestNDVI),ee.Filter.lt('norm_diff',forestDiff)).not());
  var filter3 = filter2.filter(ee.Filter.and(ee.Filter.eq('class', 1),ee.Filter.lt('norm_diff',herbDiff),ee.Filter.lt('ndvi_value', herbNDVI)).not())
  return filter3;
}
``` -->

### filterPoints
```javascript   
exports.filterPoints = function(features,filter,field){
  filter.map(function(classFilter){
    features = features.filter(ee.Filter.and(ee.Filter.eq('class', classFilter['class']),ee.Filter.gt(field, classFilter['max']),ee.Filter.lt(field,classFilter['min'])).not());
    return classFilter;
  });
  return features;
}
```

### createRFDict
```javascript
exports.createRFDict = function(RF_matrix){
  var RF_producers = RF_matrix.producersAccuracy()
  var RF_consumers = RF_matrix.consumersAccuracy()

  return ee.Dictionary({'Overall accuracy':ee.Number(RF_matrix.accuracy()).format('%.4f'),
      'Kappa':ee.Number(RF_matrix.kappa()).format('%.4f'),
      '% of class 0 accurately identified':ee.Number(RF_producers.get([0,0])).format('%.4f'),
      '% of class 1 accurately identified':ee.Number(RF_producers.get([1,0])).format('%.4f'),
      '% of class 2 accurately identified':ee.Number(RF_producers.get([2,0])).format('%.4f'),
      '% of class 3 accurately identified':ee.Number(RF_producers.get([3,0])).format('%.4f'),
      '% correct of class 0 output':ee.Number(RF_consumers.get([0,0])).format('%.4f'),
      '% correct of class 1 output':ee.Number(RF_consumers.get([0,1])).format('%.4f'),
      '% correct of class 2 output':ee.Number(RF_consumers.get([0,2])).format('%.4f'),
      '% correct of class 3 output':ee.Number(RF_consumers.get([0,3])).format('%.4f'),
  });
}
```

## VISUALIZATION
