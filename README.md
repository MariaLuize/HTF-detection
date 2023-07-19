# HYPERSALINE TIDAL FLATS DETECTION USING DEEP LEARNING OVER 37 YEARS OF
LANDSAT DATA

Hypersaline tidal flats are plane areas usually related to mangrove forests, acting as guard and buffer against rising sea
levels, and as maintainer of regional biodiversity. Such areas are primarily impacted by anthropogenic and natural activities, such as sea-salt extraction and pollution, so identifying and monitoring them is an important and challenging task. The present work uses a U-shaped Convolutional Neural Network architecture to systematically classify such formations over Landsat imagery. A large dataset containing data from 1985 to 2021 of the Brazilian Coastal Zone is used to train and evaluate our model. Experimental results show that the total area increased by 58.6 km² from 1985 to 2001, and decreased by approximately 92 km² from 2001 to 2021, representing a total reduction of ≈ 33.34 km² for the entire period. We also show that our model outperforms a related solution trained with the same dataset, achieving 70% and 86% for 1985 and 2020 respectively, against 69% and 82%.

## Visualization Script (GEE)
Visualization of final classification and statistics: https://code.earthengine.google.com/6d0837129a8cd8685336d75381ba4d4f



## Usage of the Source Script
* NECESSARY IMPORTS
```javascript
var imageVisParam = {"opacity":1,"bands":["swir1","nir","red"],"min":100,"max":143,"gamma":1},
    geometry = 
    /* color: #d63000 */
    /* shown: false */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[-44.41376933593751, -22.106248427099384],
          [-44.41376933593751, -23.55919889599254],
          [-42.75483378906251, -23.55919889599254],
          [-42.75483378906251, -22.106248427099384]]], null, false);
```
* BASIC CONFIGURATIONS
```javascript
// YEAR SELECTION
var year    = 1985;
var classID = 32;
var mosaic  = ee.Image("projects/samm/SAMM/Mosaic/"+year+"_v2");

// BASIC MOSAIC VISUALIZATION
Map.addLayer(mosaic,imageVisParam,'Mosaic '+year);

// BLANK CANVAS FOR BETTER VISUALIZATION
Map.addLayer(ee.Image(0),{palette:'FFFFFF'},'Blank',false);
```

* VISUALIZATION OF MapBiomas 6 CLASSIFICATION
```javascript
// ATTENTION: PLEASE, DO NOT MODIFY THE FOLLOWING LINES
var merge = ee.ImageCollection('projects/mapbiomas-workspace/TRANSVERSAIS/ZONACOSTEIRA6-FT')
             .filterMetadata('version','equals','3').filter(ee.Filter.eq('year',year)).max().eq(classID).unmask(0);
var displacedMergeMask = merge.focal_max(4).reproject('EPSG:4326', null, 30)
merge = merge.updateMask(displacedMergeMask.eq(1)).unmask(0)
Map.addLayer(merge,{palette:['white','red'],min:0,max:1},'Reference Mapbiomas Collection 6',false)
```

* SORT POINTS
```javascript
var points = merge.stratifiedSample(10000,'classification',geometry,30,null,1,[0,1],[5000,5000],true,1,true)
print(points.size())
print(points.first())
Map.addLayer(points,{},'Points',false)
```
* HYPERSALINE TIDAL FLATS (HTF) CLASSIFICATION
```javascript
// ATTENTION: PLEASE, DO NOT MODIFY THE FOLLOWING LINES
var HTF_Classification = ee.ImageCollection('projects/mapbiomas-workspace/TRANSVERSAIS/COLECAO7/zona-costeira')
                               .filterMetadata('version','equals','2').filter(ee.Filter.eq('year',year)).max().unmask(0)
```
* COMPARISON BETWEEN REFERENCE MAP (MapBiomas 6) WITH HTF CLASSIFICATION
```javascript
points= points.map(function(feat){
  return feat.set({'HTF_Classification':HTF_Classification.eq(classID).reduceRegion(ee.Reducer.first(),feat.geometry(),30).get('classification')})
})
print(points.first())

Map.addLayer(HTF_Classification.mask(HTF_Classification.eq(classID)),{palette:'blue'},'HTF Classification',false)
```

* CORRELATIONS
```javascript
//No-change
var noChangeP = HTF_Classification.eq(classID).and(merge.eq(1))
var noChangeN = HTF_Classification.neq(classID).and(merge.eq(0))

//Positive-Disagreement (HTF_Classification HTF -> ecosystem non-HTF)
var positiveDisagreement = HTF_Classification.eq(classID).and(merge.neq(1))

//Negative-Disagreement (HTF_Classification non-HTF -> ecosystem HTF)
var nagativeDisagreement = HTF_Classification.neq(classID).and(merge.eq(1))

var changes = ee.ImageCollection([ee.Image(1).toByte().mask(noChangeP),
                                  ee.Image(2).toByte().mask(positiveDisagreement),
                                  ee.Image(3).toByte().mask(nagativeDisagreement)
                                  ]).max().unmask(0)
Map.addLayer(changes.mask(changes.gt(0)),{palette:['000000','0000FF','00FF00','FF0000'],min:0,max:3},'Changes')
```

* STATISTICS
```javascript
var statesBR = ee.FeatureCollection('projects/solvedltda/assets/BR_UF_2021')
 var changesPerState = statesBR.map(function(feat){
      var nochangesP = noChangeP.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var nochangesN = noChangeN.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var positive = positiveDisagreement.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var negative = nagativeDisagreement.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var sigla = feat.get('sigla')
    return ee.Feature(null,{'nochangeN':nochangesN.get('classification'),'nochangeP':nochangesP.get('classification'),'positive':positive.get('classification'),'negative':negative.get('classification'),'sigla':sigla})
 })
var changesPerState_Exemple = changesPerState.first()
print('Brasilian state exemple: ', changesPerState_Exemple.get('sigla'), changesPerState_Exemple) 
Export.table.toDrive(changesPerState,'changes_per_state_'+year+'_mapbiomas','results_HTF','changes_per_state_'+year+'_mapbiomas')
```

* LOCALIZATION SET
```javascript
Map.setCenter(-43.59631,-23.01244,13)
```

* POINTS BASED STATS
```javascript
var errorM = points.errorMatrix('classification','HTF_Classification')
print('Error Matrix',errorM)
print('O.A',errorM.accuracy())
print('Kappa',errorM.kappa())
print('Consumer',errorM.consumersAccuracy())
print('Producers',errorM.producersAccuracy())
