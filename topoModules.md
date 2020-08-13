# Modules for calculating topographic variables in Google Earth Engine
[**Zach Levitt**](https://zachlevitt.github.io), '20.5</br>
[**Jeff Howarth**](https://jeffhowarth.github.io/), Associate Professor of Geography</br>
Middlebury College, Vermont, USA

These module functions are provided here as a reference for the [main tutorial](./README.md), which outlines a workflow for calculating topographic variables in Google Earth Enginel. This overview is organized in the same way as the [module script](https://code.earthengine.google.com/d0104a6e2c5a86da1ac5a03b1720d5d0), reflecting the major tasks that are involved in performing this type of analysis. 

Use this line to reference these functions in another script:
```javascript
var tt = require('users/zlevitt/chis:topoTools.js');
```

Each function and section is linked below.

* [**IMPORT and PROCESS DEM**](#import-and-process-dem)
	* [loadImageAsset](#loadimageasset)
	* [processElevationData](#processelevationdata)
* [**CALCULATE TOPOGRAPHIC VARIABLES**](#calculate-topographic-variables)
	* [calculateSlopeDegrees](#calculateslopedegrees)
	* [calculateTheobaldHLI](#calculatetheobaldhli)
	* [calculateNeighborhoodMean](#calculateneighborhoodmean)
	* [calculateNeighborhoodStdDev](#calculateneighborhoodstddev)
	* [calculateStandardizedTPI](#calculatestandardizedtpi)
	* [calculateMeanTPI](#calculatemeantpi)
	* [calculateTPI](#calculatetpi)
	* [calculateLandforms](#calculatelandforms)
	* [remapLandforms](#remaplandforms)
* [**VISUALIZATION**](#visualization)
  * [outputChart](#outputchart)
  * [makeChartStyle](#makechartstyle)

[**Return to main tutorial**](./README.md)


## IMPORT and PROCESS DEM
<!-- ```javascript
``` -->
### loadImageAsset
Input: **path** (string)</br>
Output: **image** (image)
```javascript
//Load DEM from path
  exports.loadImageAsset = function(path){
    return ee.Image(path)
  }
```

### processElevationData
Input: **image** (image), **crs** (string), **scale** (number), *optional:* **mask** (image)</br>
Output: **image** (image)
```javascript
//Reduces resolution of image 
exports.processElevationData = function(image,crs,scale,mask) {
    var output = image
    .reduceResolution({
      reducer: ee.Reducer.mean(),
      maxPixels: 1024
    })
    // Request the data at the scale and projection of the MODIS image.
    .reproject({
      crs: crs,
      scale: scale
    });
    
    if (mask !== null){
      output = output.updateMask(mask);
    }
    
    return output;
};
```
 

## CALCULATE TOPOGRAPHIC VARIABLES

### calculateSlopeDegrees
Input: **dem** (image)</br>
Output: **Slope in degrees** (image)
```javascript
//Calculate slope in degrees from elevation data
exports.calculateSlopeDegrees = function(dem){
  return ee.Terrain.slope(dem)
}
```

### calculateTheobaldHLI
Input: **dem** (image)</br>
Output: **Heat load index** (image)
```javascript
//Calculate heat load index, based on Theobald et al (2015)
exports.calculateTheobaldHLI = function(dem) {
  var aspectRadians = ee.Terrain.aspect(dem).multiply(ee.Number(0.01745329252))
  var slopeRadians = ee.Terrain.slope(dem).multiply(ee.Number(0.01745329252))
  var foldedAspect = aspectRadians.subtract(ee.Number(4.3196899)).abs().multiply(ee.Number(-1)).add(ee.Number(3.141593)).abs()
  var theobaldHLI = (slopeRadians.cos().multiply(ee.Number(1.582)).multiply(ee.Number(0.82886954044))).subtract(foldedAspect.cos().multiply(ee.Number(1.5)).multiply(slopeRadians.sin().multiply(ee.Number(0.55944194062)))).subtract(slopeRadians.sin().multiply(ee.Number(0.262)).multiply(ee.Number(0.55944194062))).add(foldedAspect.sin().multiply(ee.Number(0.607)).multiply(slopeRadians.sin())).add(ee.Number(-1.467));
  return theobaldHLI.exp().rename('hli')
};
```

### calculateNeighborhoodMean
Input: **dem** (image), **kernelRadius** (number)</br>
Output: **Mean image based on radius** (image)
```javascript
//  Calculate a neighborhood mean using a pre-defined kernel radius
exports.calculateNeighborhoodMean = function(dem, kernelRadius) {
  return dem.reduceNeighborhood({
    reducer: ee.Reducer.mean(),
    kernel: ee.Kernel.square(kernelRadius,'pixels',false),
    optimization: 'boxcar',
  });
}
```

### calculateNeighborhoodStdDev
Input: **dem** (image), **kernelRadius** (number)</br>
Output: **Standard deviation image based on radius** (image)
```javascript
//Calculate a neighborhood standard deviation using a pre-defined kernel radius    
exports.calculateNeighborhoodStdDev = function(dem, kernelRadius) {
  return dem.reduceNeighborhood({
    reducer: ee.Reducer.stdDev(),
    kernel: ee.Kernel.square(kernelRadius,'pixels',false),
    optimization: 'window',
  });
}
```

### calculateStandardizedTPI
Input: **image** (image), **meanImage** (image), **stdDevImage** (image)</br>
Output: **Standardized topographic position index**
```javascript
//Calculate standardized topographic position index, based on Theobald et al (2015)
exports.calculateStandardizedTPI = function(image, meanImage, stdDevImage) {
  return (image.subtract(meanImage)).divide(stdDevImage)
};
```

### calculateTPI
Input: **image** (image), **meanImage** (number)</br>
Output: **Topographic position index** (image)
```javascript
//Calculate topographic position index from a pre-defined mean image
exports.calculateTPI = function(image, meanImage) {
  return image.subtract(meanImage).rename('tpi')
}
```

### calculateMeanTPI
Input: **image1** (image), **image2** (number), **image3** (number)</br>
Output: **Mean topographic position index** (image) 
```javascript
//Calculate mean topographic position index, based on Theobald et al (2015)
exports.calculateMeanTPI = function(image1,image2,image3){
  return (image1.add(image2).add(image3)).divide(ee.Number(3)).rename('meanTPI')
}
```

### calculateLandforms
Input: **dem** (image), **slopeDegrees** (image), **theobaldHLI** (image), **meanTPI** (image),**tpi_270m** (image)</br>
Output: **Landforms** (image)
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
Input: **landforms** (image)</br>
Output: **Remapped landforms** (image)
```javascript
//Reclassify landforms layer to fit with visualization parameters.
exports.remapLandforms = function(landforms){
  return landforms.remap([11,12,13,14,15,21,22,23,24,31,32,33,34,41,42],[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14])
}
```

### remapLandformsSimple
Input: **landforms** (image)</br>
Output: **Remapped simplified landforms** (image)
```javascript
//Reclassify landforms layer to fit with simplified visualization parameters.
exports.remapLandformsSimple = function(landforms){
  return landforms.remap([11,12,13,14,15,21,22,23,24,31,32,33,34,41,42],[0,0,0,0,0,1,1,1,1,2,2,2,2,3,3])
}
```

## VISUALIZATION

### outputChart
Input: **landforms** (image),**geometry** (geometry), **scale** (number), **dictionary** (dictionary)</br>
Output: **Pie chart showing percent of geometry by landform class** (chart)
```javascript
exports.outputChart = function(landforms,geometry,scale,dictionary){
      //Calculate areal stats for landforms (full classes)
      var stats = ee.Image.pixelArea().addBands(landforms).reduceRegion({
        reducer: ee.Reducer.sum().group({
          groupField: 1
        }),
        geometry: geometry,
        scale: scale,
      });
      
      //reformat stats
      var statsFormatted = ee.List(stats.get('groups'))
        .map(function(el) {
          var d = ee.Dictionary(el);
          return [dictionary.get(d.get('group')), ee.Number(d.get('sum')).multiply(0.00000038610215855)];
      });
      
      //flatten dictionary
      var statsDictionary = ee.Dictionary(statsFormatted.flatten());
      
      return ui.Chart.array.values({
        array: statsDictionary.values(),
        axis: 0,
        xLabels: statsDictionary.keys()
      }).setChartType('PieChart')
        

    }
```

### makeChartStyle
Input: **title** (string), **palette** (list) </br>
Output: **Chart style object** (object)
```javascript
exports.makeChartStyle = function(title, palette){
    return {
      title: title,
      pieHole: 0.33,
      padding: '12px',
      colors: palette,
      chartArea: {
        left: 16,
        height: '75%',
        width: '100%'
      },
      titleTextStyle: {
        color: '#000000',
        fontSize: 13,
        padding:'5px',
      },
      legend: {
        textStyle: {
            alignment: 'end',
            color: 'black',
            fontName: 'Helvetica',
            fontSize: 18
            },
        position: 'left'
        }
    }
  }
```

