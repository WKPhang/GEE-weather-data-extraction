# Google Earth Engine Script: Weather data extraction

## Objective 1
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

## Objective 2 
To calculate the daily precipitation for each points according to given dates
```
// Load your shapefile (ensure it is uploaded as an Asset)
var points = ee.FeatureCollection('users/your_username/your_shapefile');

// Function to parse and format the date, handling both DD/MM/YYYY and DD/M/YYYY formats
function formatDate(feature) {
  var dateStr = ee.String(feature.get('date'));
  
  // Check if the month part has a single or double digit
  var parts = dateStr.split('/');
  var day = parts.get(0);
  var month = parts.get(1);
  var year = parts.get(2);
  
  // Format month to ensure it is two digits
  var formattedMonth = ee.String(month).length().eq(1)
    ? ee.String('0').cat(month)
    : month;
  
  // Combine parts into a proper date string
  var formattedDate = ee.String(year).cat('-').cat(formattedMonth).cat('-').cat(day);
  
  return feature.set('formatted_date', formattedDate);
}

// Apply the formatting function to all features
points = points.map(formatDate);

// Load the CHIRPS daily precipitation data
var chirps = ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY");

// Function to extract precipitation data for each point based on the date
function extractPrecipitation(feature) {
  var date = ee.Date(feature.get('formatted_date'));
  var image = chirps.filterDate(date, date.advance(1, 'day')).first();
  
  // Get the precipitation value at the point location
  var precipitation = image.reduceRegion({
    reducer: ee.Reducer.first(),
    geometry: feature.geometry(),
    scale: 5566  // Use the native resolution of CHIRPS
  }).get('precipitation');
  
  // Return the feature with the precipitation value
  return feature.set('precipitation', precipitation);
}

// Map the extraction function over all points
var pointsWithPrecipitation = points.map(extractPrecipitation);

// Export the results as a CSV or to your Google Drive
Export.table.toDrive({
  collection: pointsWithPrecipitation,
  description: 'Precipitation_Extraction',
  fileFormat: 'CSV'
});

// Optional: Print results to console for quick inspection
print(pointsWithPrecipitation);
```
