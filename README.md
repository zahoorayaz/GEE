// Import boundaries from asset
var AOI = ee.FeatureCollection('projects/centering-timer-425613-e4/assets/jhang');

// Set map center to the AOI for making sure we have the correct study area
Map.centerObject(AOI, 9);

// Load the Sentinel-1 image collection.
var sentinel1 = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(AOI)
  .filterDate('2023-01-01', '2023-12-31')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.eq('instrumentMode', 'IW'));

// Select the VV polarization and take the median.
var vv = sentinel1.select('VV').median().clip(AOI);

// Display the VV image.
Map.addLayer(vv, {min: -25, max: 0}, 'VV polarization');

// Create training data: Define land cover classes and points.
var urban = ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([72.3, 31.3]), {'class': 0}),
  ee.Feature(ee.Geometry.Point([72.35, 31.35]), {'class': 0}),
  // Add more points as needed
]);

var water = ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([72.4, 31.2]), {'class': 1}),
  ee.Feature(ee.Geometry.Point([72.45, 31.25]), {'class': 1}),
  // Add more points as needed
]);

var vegetation = ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([72.2, 31.1]), {'class': 2}),
  ee.Feature(ee.Geometry.Point([72.25, 31.15]), {'class': 2}),
  // Add more points as needed
]);

// Merge all classes into one FeatureCollection.
var trainingPoints = urban.merge(water).merge(vegetation);

// Sample the input imagery to get a FeatureCollection of training data.
var training = vv.sampleRegions({
  collection: trainingPoints,
  properties: ['class'],
  scale: 9
});

// Train a Random Forest classifier with default parameters.
var classifier = ee.Classifier.smileRandomForest(10).train({
  features: training,
  classProperty: 'class',
  inputProperties: ['VV']
});

// Classify the image.
var classified = vv.classify(classifier);

// Define a palette for the classes.
var palette = [
  'red',    // urban
  'blue',   // water
  'green'   // vegetation
];

// Add the classified image to the map.
Map.addLayer(classified, {min: 0, max: 2, palette: palette}, 'Land Cover');

// Export the classified image to your Google Drive.
Export.image.toDrive({
  image: classified,
  description: 'Sentinel1_Classification_Jhang',
  scale: 9,
  region: AOI.geometry().bounds(),
  folder: 'JHANG_BD',
  fileNamePrefix: 'Sentinel1_Classification_Jhang',
  fileFormat: 'GeoTIFF'
});
