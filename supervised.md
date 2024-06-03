var hathazari = ee.Geometry.Rectangle(91.7798, 22.4287, 91.9025, 22.5336);

var sentinel2 = ee.ImageCollection('COPERNICUS/S2')
  .filterBounds(hathazari)
  .filterDate('2023-01-01', '2023-12-31')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10))
  .median();

var vegetation= ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([91.8006, 22.4900]), {landcover: 0})
  ]);
var settlement = ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([91.7999, 22.4901]), {landcover: 1})
  ]);
var water= ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([91.8012, 22.4910]), {landcover: 2}) 
]);

var trainingData = settlement.merge(vegetation).merge(water);
print(trainingData)


var bands = ['B2', 'B3', 'B4', 'B8']; 
var training = sentinel2.select(bands).sampleRegions({
  collection: trainingData,
  properties: ['landcover'],
  scale: 10
});

var classifier = ee.Classifier.smileRandomForest(10).train({
  features: training,
  classProperty: 'landcover',
  inputProperties: bands
});

var classified = sentinel2.select(bands).classify(classifier);

Export.image.toDrive({
  image: classified,
  description: 'Hathazari_ClassifiedImage',
  scale: 10,
  region: hathazari,
  folder: "supervised",
  fileFormat: 'GeoTIFF',
  maxPixels: 1e9
});

Map.centerObject(hathazari, 12);

var visParams = {
  min: 0,
  max: 2, 
  pallette: ['red','green','blue']
};

Map.addLayer(hathazari, {bands: ['B4', 'B3', 'B2'], min: 0, max:3000})
Map.addLayer(classified,visParams, 'LULC Classification');

var area = ee.Image.pixelArea().addBands(classified)
  .reduceRegion({
    geometry: hathazari,
    reducer: ee.Reducer.sum().group({
      groupField: 1,
      groupName: 'landcover'
    }),
    scale: 10,
    bestEffort: true
  }).get("groups");

print('Area of each landcover class:', area);
