# Google Earth Engine Script: Weather data extraction using polygons

## Objectives
To calculate the average precipitation for each area of interest (in the form of polygon)

```
// Data Imports & Global variables
var my_polygon = ee.FeatureCollection("path/to/imported_polygon_shapefile");

// Visualize the polygon shapefile
Map.addLayer(my_polygon,{} ,'My polygons', true);
Map.centerObject(my_polygon,  3);

// Define the date range for extracting data from 01/01/2012 to 31/12/2023
var startDate = ee.Date('2012-01-01');
var endDate = ee.Date('2023-01-01');

// Load the CHIRPS data collection.
var chirps_data = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY');

// Filter the data collection for the desired date range
// select the band
var selectedBand = chirps_data
  .filterDate(startDate, endDate)
  .select('precipitation');

var list_selectedBand = selectedBand.toList(selectedBand.size());

print(list_selectedBand);

var selectedBand_Vis = {
  min: -300.0,
  max: 400.0,
  palette: [
    '1a3678', '2955bc', '5699ff', '8dbae9', 'acd1ff', 'caebff', 'e5f9ff',
    'fdffb4', 'ffe6a2', 'ffc969', 'ffa12d', 'ff7c1f', 'ca531a', 'ff0000',
    'ab0000'
  ],
};

Map.addLayer(selectedBand, selectedBand_Vis, 'Precipitation');

// The pixel values have a scale factor of 0.1
// We must multiply the pixel values with the scale factor
// to get the temperature values in °C
var selectedBandScaled = selectedBand.map(function(image) {
  return image.multiply(1)
    .copyProperties(image,['system:time_start']);
});

// Make monthly means
var monthlyMeans = ee.ImageCollection.fromImages(

  // Generate a list of years from 2012 to 2022 (inclusive)
  ee.List.sequence(2012, 2022, 1)
    .map(function(year) {

      // Create a list of months (1 to 12) for the current year
      var months = ee.List.sequence(1, 12);

      // Apply a function to each month to create a monthly mean image
      return months.map(function(month) {
        var start = ee.Date.fromYMD(year, month, 1);
        var end = start.advance(1, 'month');  // Move to the end of the month

        // Calculate monthly mean for the scaled band
        var monthlyMeanImage = selectedBandScaled
          .filterDate(start, end)
          .mean()
          .set('year', year)
          .set('month', month);

        return monthlyMeanImage;
      });
    })
  // Flatten the nested list into a single collection of monthly images
  .flatten()
)

// filter out years without images
  .filter(ee.Filter.listContains('system:band_names', 'precipitation'));
print(monthlyMeans)

// Extract values for each polygon and combine into a single collection
var mapped = monthlyMeans.map(function(image){
  var allFeatures = image.reduceRegions({
    collection: my_polygon,
    reducer: ee.Reducer.mean(),
    scale: 5566
  }).map(function(feature){
    return feature
      .set('month', image.get('month'))
      .set('year', image.get('year'))
  });
  return allFeatures
}).flatten();

// Export the feature collection to Google Drive
Export.table.toDrive({
  collection: mapped,
  description: 'CHIRPS_Precip_Average_by_Month_2012_to_2022',
  folder: 'GEE_exports',
  fileFormat: 'CSV'
});
```
