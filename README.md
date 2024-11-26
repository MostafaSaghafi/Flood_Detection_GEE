# Flood Detection and Flood Area using GEE
Detecting the flood and calculate the flooded area 
--------

This code is for detecting flood inundation and water bodies using Sentinel-1 SAR (Synthetic Aperture Radar) data in Google Earth Engine (GEE). It involves several steps, including data filtering, speckle noise reduction, flood and water body detection, visualization, and export of results. Let’s break it down line by line:

---

### **1. Filtering Sentinel-1 SAR Data**
```javascript
var collection = ee.ImageCollection('COPERNICUS/S1_GRD')
        .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
        .filter(ee.Filter.eq('instrumentMode', 'IW'))
        .filter(ee.Filter.or(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'),ee.Filter.eq('orbitProperties_pass', 'ASCENDING')))
        .select('VV'); // Only select the 'VV' band
```
- **`ee.ImageCollection('COPERNICUS/S1_GRD')`:** Accesses Sentinel-1 Ground Range Detected (GRD) data.
- **`filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))`:** Filters images with VV polarization (vertical transmit and receive).
- **`filter(ee.Filter.eq('instrumentMode', 'IW'))`:** Selects images captured in Interferometric Wide (IW) mode.
- **`filter(ee.Filter.or(...))`:** Filters images based on orbit direction (ascending or descending).
- **`select('VV')`:** Selects the VV polarization band for analysis.


---

### **2. Filtering Images Before and After Flood**
```javascript
var before = collection.filter(ee.Filter.date('2015-01-01', '2015-12-29'))
  .filterBounds(dt).max()
var after = collection.filter(ee.Filter.date('2015-02-20', '2015-04-01'))
  .filterBounds(dt).min()
```
- **`filter(ee.Filter.date(...))`:** Filters images within specific date ranges (`before` and `after` the flood event).
- **`filterBounds(dt)`:** Filters images within a specific region of interest (`dt` is the study area geometry).
- **`.max()` and `.min()`:** Select the brightest (`max`) and darkest (`min`) images within the filtered collections.

---

### **3. Clipping and Preprocessing Images**
```javascript
var before_image = before.select('VV').clip(dt);
var after_image = after.select('VV').clip(dt);
```
- **`select('VV')`:** Ensures only the VV band is used.
- **`clip(dt)`:** Clips the images to the study area (`dt`).

---

### **4. Speckle Noise Filtering**
```javascript
var before_filtered = ee.Image(toDB(RefinedLee(toNatural(before_image))))
var after_filtered = ee.Image(toDB(RefinedLee(toNatural(after_image))))
```
- **Speckle Noise:** SAR images often contain speckle noise, which needs to be reduced.
- **`toNatural`:** Converts the image from decibels (dB) to natural units.
- **`RefinedLee`:** Applies the Refined Lee filter, a speckle noise reduction technique.
- **`toDB`:** Converts the filtered image back to dB.

---

### **5. Flood and Water Detection**
```javascript
var flood = before_filtered.gt(-19).and(after_filtered.lt(-19));
var flood_mask = flood.updateMask(flood.eq(1));
```
- **Flood Detection:** Compares the radar backscatter values before and after the flood:
  - **`before_filtered.gt(-19)`:** Identifies pixels with backscatter > -19 dB before the flood.
  - **`after_filtered.lt(-19)`:** Identifies pixels with backscatter < -19 dB after the flood.
  - **`and(...)`:** Combines these conditions to detect flooded areas.
- **`updateMask`:** Masks pixels that do not meet the flood condition.

```javascript
var water = before_filtered.lt(-14).and(after_filtered.lt(-14));
var water_mask = water.updateMask(water.eq(1));
```
- **Water Body Detection:** Identifies permanent water bodies with backscatter < -14 dB in both images.

---

### **6. Visualization**
```javascript
Map.centerObject(dt);
Map.addLayer(before_image,{min:-25,max:0},'before')
Map.addLayer(after_image,{min:-25,max:0},'after')
Map.addLayer(flood_mask,{palette:['Red']},'Flood_Inundation')
Map.addLayer(water_mask,{palette:['Yellow']},'Water')
```
- **`Map.centerObject(dt)`:** Centers the map on the study area.
- **`Map.addLayer`:** Adds layers for visualization:
  - **Before and After Images:** Display backscatter values.
  - **Flood Mask:** Highlighted in red.
  - **Water Mask:** Highlighted in yellow.

---

### **7. Flood Area Calculation**
```javascript
var stats = flood_mask.multiply(ee.Image.pixelArea()).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: dt,
  scale: 10,
  maxPixels: 1e13,
  tileScale: 16
})
```
- **`flood_mask.multiply(ee.Image.pixelArea())`:** Calculates the area of flooded pixels.
- **`reduceRegion`:** Sums the pixel areas within the study region (`dt`).
- **`scale: 10`:** Uses a 10-meter resolution.

```javascript
var flood_area = ee.Number(stats.get('sum')).divide(10000).round();
print('Flooded Area (Ha)', flood_area)
```
- **Flood Area:** Converts the result to hectares (`divide(10000)`) and rounds the value.

---

### **8. Adding Legend**
```javascript
var legend = ui.Panel({ style: { position: 'bottom-left', padding: '8px 15px' } });
var legendTitle = ui.Label({ value: 'Legend', style: { fontWeight: 'bold', fontSize: '18px' } });
legend.add(legendTitle);
var makeRow = function(color, name) { ... };
var palette = ['Red', 'Yellow'];
var names = ['Flood Inundation','Water body'];
for (var i = 0; i < 2; i++) { legend.add(makeRow(palette[i], names[i])); }
Map.add(legend);
```
- **Legend:** Creates a UI panel with colors (`Red` for flood, `Yellow` for water) and labels.

---

### **9. Adding Map Title**
```javascript
var title = ui.Panel({ style: { position: 'top-center', padding: '8px 15px' } });
var mapTitle = ui.Label({ value: 'Flood Inundation in KamalKhan Dam, Afghanestan', style: { fontWeight: 'bold', fontSize: '13px' } });
title.add(mapTitle);
Map.add(title);
```
- **Map Title:** Adds a title to the map.

---

### **10. Exporting the Combined Mask**
```javascript
var combined_mask = flood_mask.multiply(1).add(water_mask.multiply(2));
Export.image.toDrive({
  image: combined_mask,
  description: 'Flood_Inundation_and_Water_Mask',
  scale: 10,
  region: dt,
  fileFormat: 'GeoTIFF',
  maxPixels: 1e13
});
```
- **Combined Mask:** Combines flood and water masks into one layer (flood = 1, water = 2).
- **Export:** Exports the combined mask as a GeoTIFF file to Google Drive.

---

### **11. Speckle Filtering Functions**
```javascript
function toNatural(img) { ... }
function toDB(img) { ... }
function RefinedLee(img) { ... }
```
- **`toNatural` and `toDB`:** Convert between dB and natural units.
- **`RefinedLee`:** Implements the Refined Lee speckle filter for noise reduction.

---

### **Summary**
This script uses Sentinel-1 SAR data to detect and analyze flood inundation and water bodies in a specified region. It includes data preprocessing, speckle filtering, flood/water detection, visualization, and export of results.
