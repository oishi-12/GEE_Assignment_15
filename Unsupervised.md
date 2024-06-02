var roi = 
    /* color: #d63000 */
    /* shown: false */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[91.78680175925813, 22.52211352616969],
          [91.78680175925813, 22.482782804052437],
          [91.8461965956839, 22.482782804052437],
          [91.8461965956839, 22.52211352616969]]], null, false);


var landsat9 = ee.ImageCollection("LANDSAT/LC09/C02/T1_L2")
                .filterBounds(roi)
                .filterDate("2023-01-01", "2023-12-31")
                .filter(ee.Filter.lt('CLOUD_COVER', 20))
                .median()
                .clip(roi);

var bands = ["SR_B2", "SR_B3", "SR_B4", "SR_B5"];
var image = landsat9.select(bands);

Map.centerObject(roi, 10);
Map.addLayer(image, {bands: ['SR_B4', 'SR_B3', 'SR_B2'], min: 0, max: 3000}, 'Landsat 9 RGB');

print(landsat9);

var training = image.sample({
  region: roi,
  scale: 30,
  numPixels: 5000
});
print(training);

var clusterer = ee.Clusterer.wekaKMeans(5).train(training);

var result = image.cluster(clusterer);
print(result)
Map.addLayer(result.randomVisualizer(), {}, "K means-Clusters");



Export.image.toDrive({
  image: result,
  description: "unsup_data",
  scale: 10,
  folder: "unsupervised",
  
  region:roi,
  maxPixels: 1e13
});
