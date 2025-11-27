import ee, geemap



ee.Initialize(project='')



china\_adm1 = ee.FeatureCollection("FAO/GAUL/2015/level1")

province\_boundary = china\_adm1.filter(

&nbsp;   ee.Filter.eq('ADM1\_NAME', 'Province name ')

).geometry()

region = Province\_boundary

def maskS2clouds(image):

&nbsp;   qa = image.select('QA60')

&nbsp;   cloudBitMask = 1 << 10

&nbsp;   cirrusBitMask = 1 << 11

&nbsp;   mask = qa.bitwiseAnd(cloudBitMask).eq(0).And(

&nbsp;       qa.bitwiseAnd(cirrusBitMask).eq(0)

&nbsp;   )

&nbsp;   return image.updateMask(mask).divide(10000)

s2\_collection = ee.ImageCollection('COPERNICUS/S2\_SR\_HARMONIZED') \\

&nbsp;   .filterDate('2024-01-01', '2025-11-01') \\

&nbsp;   .filterBounds(region) \\

&nbsp;   .map(maskS2clouds)

s2\_median = s2\_collection.median().clip(region)

ndvi = s2\_median.normalizedDifference(\['B8', 'B4']).rename('NDVI')

ndbi = s2\_median.normalizedDifference(\['B11', 'B8']).rename('NDBI')

mndwi = s2\_median.normalizedDifference(\['B3', 'B11']).rename('MNDWI')

swir\_nir\_ratio = s2\_median.select('B11').divide(s2\_median.select('B8')).rename('SWIR\_NIR')

glcm = s2\_median.select('B8').multiply(255).toUint8().glcmTexture(size=3)

texture\_features = glcm.select(\[

&nbsp;   'B8\_contrast',

&nbsp;   'B8\_ent',

&nbsp;   'B8\_var',

&nbsp;   'B8\_idm',

&nbsp;   'B8\_diss'

]).rename(\[

&nbsp;   'Contrast',

&nbsp;   'Entropy',

&nbsp;   'Variance',

&nbsp;   'Homogeneity',

&nbsp;   'Dissimilarity'

])



viirs\_2024 = ee.ImageCollection("NOAA/VIIRS/DNB/MONTHLY\_V1/VCMSLCFG") \\

&nbsp;   .filterDate('2024-01-01', '2025-11-01') \\

&nbsp;   .select('avg\_rad') \\

&nbsp;   .mean() \\

&nbsp;   .clip(region) \\

&nbsp;   .rename('Nightlight')

feature\_image = ee.Image.cat(\[

&nbsp;   s2\_median.select(\['B2','B3','B4','B8','B11','B12']),  

&nbsp;   ndvi, ndbi, mndwi, swir\_nir\_ratio,                   

&nbsp;   texture\_features,                                    

&nbsp;   viirs\_2024                                           

]).clip(region)

task = ee.batch.Export.image.toAsset(

&nbsp;   image = feature\_image,

&nbsp;   description = 'Province\_Feature\_Image',

&nbsp;   assetId = 'projects/../assets/Province\_Feature\_Image',

&nbsp;   region = region,

&nbsp;   scale = 10,     

&nbsp;   maxPixels = 1e13

)



task.start()

import ee

import geemap

import pandas as pd

csv\_path\_positive = Actual path

try:

&nbsp;   poi\_fc = geemap.csv\_to\_ee(csv\_path\_positive, latitude='wgs84\_lat', longitude='wgs84\_lon')

&nbsp;   positive\_samples = poi\_fc.map(lambda f: f.set('label', 1))

&nbsp;   num\_positive = positive\_samples.size().getInfo()

china\_adm1 = ee.FeatureCollection("FAO/GAUL/2015/level1")

negative\_samples = ee.FeatureCollection(\[])

&nbsp;   def filter\_points(points, mask):

&nbsp;       return points.map(lambda f: f.set('mask\_val', mask.reduceRegion(

&nbsp;           reducer=ee.Reducer.first(),

&nbsp;           geometry=f.geometry(),

&nbsp;           scale=30

&nbsp;       ).get('NDVI'))).filter(ee.Filter.eq('mask\_val', 1))



&nbsp;   ndvi = feature\_image.normalizedDifference(\['B8', 'B4']).rename('NDVI')

&nbsp;   ndbi = feature\_image.normalizedDifference(\['B11', 'B8']).rename('NDBI')

&nbsp;   nightlight = feature\_image.select('Nightlight')



&nbsp;   loose\_mask = ndvi.lt(0.5).And(ndbi.gt(-0.3)).And(nightlight.gt(0.05)).selfMask()

&nbsp;   num\_candidates = num\_positive \* 50

&nbsp;   raw\_points = ee.FeatureCollection.randomPoints(region=jiangsu\_boundary, points=num\_candidates, seed=42)

&nbsp;   filtered\_points = filter\_points(raw\_points, loose\_mask)

&nbsp;   filtered\_count = filtered\_points.size().getInfo()



&nbsp;   if filtered\_count >= num\_positive:

&nbsp;       negative\_samples = filtered\_points.randomColumn('rand', seed=42).limit(num\_positive)

&nbsp;   else:

&nbsp;       backup\_mask = ndvi.lt(0.4).And(nightlight.gt(0.01)).selfMask()

&nbsp;       if filtered\_count < num\_positive \* 0.5:

&nbsp;           raw\_points = ee.FeatureCollection.randomPoints(region=jiangsu\_boundary, points=num\_positive \* 100, seed=43)

&nbsp;       filtered\_points\_backup = filter\_points(raw\_points, backup\_mask)

&nbsp;       filtered\_backup\_count = filtered\_points\_backup.size().getInfo()

&nbsp;       if filtered\_backup\_count > 0:

&nbsp;           negative\_samples = filtered\_points\_backup.randomColumn('rand', seed=42).limit(min(num\_positive, filtered\_backup\_count))

&nbsp;       else:

&nbsp;           negative\_samples = raw\_points.randomColumn('rand', seed=42).limit(num\_positive)



&nbsp;   negative\_samples = negative\_samples.map(lambda f: f.set('label', 0))

all\_samples = positive\_samples.merge(negative\_samples)

if all\_samples.size().getInfo() > 1:

&nbsp;   all\_samples\_random = all\_samples.randomColumn('random', seed=42)

&nbsp;   training\_points = all\_samples\_random.filter(ee.Filter.lt('random', 0.7))

&nbsp;   validation\_points = all\_samples\_random.filter(ee.Filter.gte('random', 0.7))

else:

&nbsp;   training\_points = validation\_points = ee.FeatureCollection(\[])



classifier = None

if training\_points.size().getInfo() > 0:

&nbsp;   training\_data = feature\_image.sampleRegions(collection=training\_points, properties=\['label'], scale=10, tileScale=4)

&nbsp;   classifier = ee.Classifier.smileRandomForest(numberOfTrees=200, seed=42).train(

&nbsp;       features=training\_data,

&nbsp;       classProperty='label'

&nbsp;   ).setOutputMode('PROBABILITY')

&nbsp;   print("Model training completed.")

else:

&nbsp;   print("The training set is empty and the model has not been trained.")

if validation\_points.size().getInfo() > 0 and classifier:

&nbsp;   validation\_classifier = classifier.setOutputMode('CLASSIFICATION')

&nbsp;   validation\_data = feature\_image.sampleRegions(collection=validation\_points, properties=\['label'], scale=10)

&nbsp;   validation\_classified = validation\_data.classify(validation\_classifier)

&nbsp;   cm = validation\_classified.errorMatrix('label', 'classification')

&nbsp;   cm\_info = cm.getInfo()

&nbsp;   print("Model Accuracy Evaluation Results:")

&nbsp;   print("Confusion Matrix:", cm\_info)

&nbsp;   print(f"Overall accuracy rate: {cm.accuracy().getInfo():.4f}")

&nbsp;   TN, FP, FN, TP = cm\_info\[0]\[0], cm\_info\[0]\[1], cm\_info\[1]\[0], cm\_info\[1]\[1]

&nbsp;   precision = TP / (TP + FP) if (TP + FP) > 0 else 0

&nbsp;   recall = TP / (TP + FN) if (TP + FN) > 0 else 0

&nbsp;   f1\_score = 2 \* precision \* recall / (precision + recall) if (precision + recall) > 0 else 0

&nbsp;   print(f"Precision={precision:.4f}, Recall={recall:.4f}, F1={f1\_score:.4f}")

&nbsp;   prob\_image\_raw = feature\_image.classify(classifier).clip(jiangsu\_core)

&nbsp;   prob\_image = prob\_image\_raw.rename('probability')

&nbsp;   uncertain\_mask = prob\_image.select('probability').gte(0.3).And(prob\_image.select('probability').lte(0.7)).selfMask()

&nbsp;   uncertain\_points = ee.FeatureCollection.randomPoints(

&nbsp;       region=uncertain\_mask.geometry(),

&nbsp;       points=100,

&nbsp;       seed=42,

&nbsp;       maxError=10

&nbsp;   )



&nbsp;   def add\_wgs84\_lat\_lon(feature):

&nbsp;       coords = feature.geometry().coordinates()

&nbsp;       return feature.set({

&nbsp;           'wgs84\_lon': coords.get(0),

&nbsp;           'wgs84\_lat': coords.get(1),

&nbsp;           'label': -1

&nbsp;       })



&nbsp;   uncertain\_points\_with\_latlon = uncertain\_points.map(add\_wgs84\_lat\_lon)



&nbsp;   def add\_probability(feature):

&nbsp;       prob = prob\_image.reduceRegion(

&nbsp;           reducer=ee.Reducer.first(),

&nbsp;           geometry=feature.geometry(),

&nbsp;           scale=10

&nbsp;       ).get('probability')

&nbsp;       return feature.set('probability', prob)



&nbsp;   final\_samples = uncertain\_points\_with\_latlon.map(add\_probability)

&nbsp;       export\_path = "Actual truth"  

&nbsp;   try:

&nbsp;       geemap.ee\_export\_vector(

&nbsp;           final\_samples,

&nbsp;           filename=export\_path,

&nbsp;           selectors=\['wgs84\_lon', 'wgs84\_lat', 'probability', 'label']

&nbsp;       )





