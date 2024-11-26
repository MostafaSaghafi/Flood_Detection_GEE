# Flood Detection and Flood Area using GEE
Detecting the flood and calculate the flooded area 
--------

This code calculates and displays flood inundation and permanent water areas using Sentinel-1 SAR imagery within a specified region of interest. It applies speckle filtering and thresholding to identify flooded and waterlogged areas and calculates the flooded area.

--------

**1. Data Acquisition and Preprocessing:**

```javascript
var collection = ee.ImageCollection('COPERNICUS/S1_GRD')
        .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
        .filter(ee.Filter.eq('instrumentMode', 'IW'))
        .filter(ee.Filter.or(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'),ee.Filter.eq('orbitProperties_pass', 'ASCENDING')))
        .select('VV'); 
```
This section filters the Sentinel-1 GRD image collection:
* `filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))`: Selects images with VV polarization (vertical transmit, vertical receive).  VV polarization is often preferred for flood mapping due to its sensitivity to water surfaces.
* `filter(ee.Filter.eq('instrumentMode', 'IW'))`: Filters for Interferometric Wide swath mode, the standard acquisition mode for Sentinel-1.
* `filter(ee.Filter.or(...))`: Includes both ascending and descending orbits. This increases the temporal frequency of observations.
* `select('VV')`: Selects only the 'VV' band from the images.

**2. Defining Date Ranges and Region:**

The commented-out date ranges likely represent different periods for analysis.  The variable `dt` (not defined in the provided code snippet) represents the region of interest (geometry). It's crucial for filtering and analysis.

```javascript
var before = collection.filter(ee.Filter.date('2015-01-01', '2020-12-29'))
  .filterBounds(dt).max()
var after = collection.filter(ee.Filter.date('2015-02-20', '2020-04-01'))
  .filterBounds(dt).min()
```

* `filter(ee.Filter.date(...))`: Filters the collection based on the specified date ranges.  `before` uses a period presumably representing pre-flood conditions.  `after` represents the period during or post-flood.
* `.filterBounds(dt)`: Filters images that intersect the region of interest defined by `dt`.  This ensures that only relevant images are processed.
* `.max()` / `.min()`: Composites the images within the filtered collections.  `.max()` takes the maximum value for each pixel across all images in the 'before' period, which is suitable for finding stable backscatter conditions pre-flood. Conversely, `.min()` selects the minimum value for the 'after' period, which helps highlight areas of low backscatter from inundated surfaces.



**3. Image Preprocessing and Filtering:**

```javascript
var before_image = before.select('VV').clip(dt);
var after_image = after.select('VV').clip(dt);

var before_filtered = ee.Image(toDB(RefinedLee(toNatural(before_image))))
var after_filtered = ee.Image(toDB(RefinedLee(toNatural(after_image))))
```

* `.select('VV').clip(dt)`: Selects the 'VV' band and clips the images to the region of interest, `dt`.  This step is repeated for both 'before' and 'after' images.
* `toNatural(...)`: Converts the image from decibels (dB) to natural units.  This is necessary for the Refined Lee filter.
* `RefinedLee(...)`: Applies the Refined Lee speckle filter to reduce noise in the SAR imagery while preserving edges and features. This function is defined later in the code.
* `toDB(...)`: Converts the filtered image back to decibels.


**4. Flood and Water Mapping:**

```javascript
var flood = before_filtered.gt(-19).and(after_filtered.lt(-19));
var flood_mask = flood.updateMask(flood.eq(1));

var water = before_filtered.lt(-14).and(after_filtered.lt(-14));
var water_mask = water.updateMask(water.eq(1));
```

* `before_filtered.gt(-19).and(after_filtered.lt(-19))`: Creates a flood mask by identifying pixels where the 'before' image value is greater than -19 dB and the 'after' image value is less than -19 dB. This threshold-based approach assumes that flooded areas will show a decrease in backscatter.
* `flood.updateMask(flood.eq(1))`: Masks the flood image, keeping only pixels where the flood condition is true (value 1).
* Similarly, the code creates a water mask using a threshold of -14 dB, assuming persistent water bodies will consistently have low backscatter in both periods.

**5. Visualization:**

```javascript
Map.centerObject(dt);
Map.addLayer(before_image,{min:-25,max:0},'before')
Map.addLayer(after_image,{min:-25,max:0},'after')
// ... other layers
Map.addLayer(flood_mask,{palette:['Red']},'Flood_Inundation')
Map.addLayer(water_mask,{palette:['Yellow']},'Water')
```
This section displays the images on the map:  It centers the map on the region of interest (`dt`) and adds the 'before' and 'after' images, the flood mask (in red), and the water mask (in yellow).


**6. Flood Area Calculation:**

```javascript
var stats = flood_mask.multiply(ee.Image.pixelArea()).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: dt,
  scale: 10,
  // ... other options
});

print(stats);
var flood_area = ee.Number(stats.get('sum')).divide(10000).round();
print('Flooded Area (Ha)', flood_area)
```

* `flood_mask.multiply(ee.Image.pixelArea())`: Multiplies the flood mask by the pixel area to get the area of each flooded pixel.
* `reduceRegion(...)`: Calculates the sum of the flooded pixel areas within the region of interest (`dt`).
* The calculated flooded area is then divided by 10000 to convert it from square meters to hectares and rounded.

**7. Legend and Title Creation:**

The code then creates a legend and a title for the map using the `ui.Panel` and `ui.Label` objects. It adds colored boxes and descriptions for the different classes (Flood Inundation and Water).

**8. Speckle Filtering Functions:**

The final part of the code defines the `toNatural`, `toDB`, and `RefinedLee` functions, which were used earlier for image preprocessing.  The `RefinedLee` function implements the refined Lee speckle filter algorithm. It uses kernels and neighborhood operations to estimate gradients and directions in the image and smooth the pixel values based on these directional statistics.


**9. Exporting the Combined Mask:**

Finally, the code combines the flood and water masks and exports the combined mask as a GeoTIFF to Google Drive. The `combined_mask` is created by multiplying the `flood_mask` by 1 and the `water_mask` by 2 and adding them together. This assigns unique values to each class in the combined mask (1 for flood, 2 for water).



Remember to define the `dt` variable with your region of interest to run the code effectively.
