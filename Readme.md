This is a more abstract and readable version of the logic used to stratify initial classification based various rules using raster operations.

# Imports and other Header Information
```python
import pandas as pd
import arcpy
from arcpy.sa import *
from arcgis.gis import GIS
import arcgis
import os
```

```
rulesPath = MANUALLY_INPUTTED_RULE_PATH

ruleSteps = pd.ExcelFile(rulesPath).sheet_names

aprx = arcpy.mp.ArcGISProject("CURRENT")

mapView = aprx.listMaps()[0]

layers = mapView.listLayers()

rasters = {layer.name: Raster(arcpy.Describe(layer).catalogPath) 
           for layer in layers if layer.isRasterLayer}

arcpy.env.mask = rasters['unet']

outputPath = MANUALLY_INPUTTED_OUTPUT_PATH
```
# Functions

```python
def ingestRules(rulesPath, ruleStep, ruleColumns):
    try:
        df = pd.read_excel(rulesPath, sheet_name = ruleStep, usecols = ruleColumns)
        return df
    except FileNotFoundError:
        print(f'Error: File not found at {rulesPath}')
        return None
    except ValueError:
        print(f'Error: One or more column names are invalid in the Rules file')
        return None
    except Exception as e:
        print(f'An unexpected error occured: {e}')
        return None
    
def calculate(init, inter, step):
    firstRasterSize = int(
        arcpy.GetRasterProperties_management(
            init, 
            "MAXIMUM").getOutput(0))
    
    secondRasterSize = int(
        arcpy.GetRasterProperties_management(
            inter, 
            "MAXIMUM").getOutput(0))
    
    reclassPath = os.path.join(outputPath, 'rasterCalc' + str(step) + '.tif')
    
    if firstRasterSize > secondRasterSize:
        (inter * (10**int(math.ceil(math.log10(firstRasterSize)))) 
         + init).save(reclassPath)
        
        return reclassPath, inter, init, firstRasterSize # smaller, larger
    else:
        (init * (10**int(math.ceil(math.log10(secondRasterSize)))) 
         + inter).save(reclassPath)
        
        return reclassPath, init, inter, secondRasterSize # smaller, larger
        
def reclassify(df, toReclass, codes, larger, step):
    remapValues = 0

    scalingFactor = 10**int(math.ceil(math.log10(larger)))

    remapValues = RemapValue(
        [[row[codes[0]] * scalingFactor + row[codes[1]], 
          row[codes[2]]] 
         for index, row in df.iterrows()]
    )
    
    reclassifiedPath = os.path.join(outputPath, 'reclassified' + str(step) + '.tif')
    
    Reclassify(os.path.join(outputPath, 'rasterCalc' + str(step) + '.tif'),
           "VALUE",
           RemapValue([[row[codes[0]] * scalingFactor + row[codes[1]], 
                        row[codes[2]]] for index, row in df.iterrows()])).save(
                reclassifiedPath
           )
    
    return reclassifiedPath

def stratify(rules, currStep = 0, reclassifiedPath = None):
    step, drop, init, inter, operation = rules[currStep].split('-')
    
    stepNum = int(step[-1]) # for example, get last character of string 'step1'

    codes = [init.upper() + 'code', #INIT_NAMEcode
             inter.upper() + 'code', #INTER_NAMEcode
             init.upper()[:-1] + str(stepNum + 1) + 'code' if len(init) == 5
             else init.upper() + str(stepNum + 1) + 'code'] #INIT_NAME2Acode or INIT_NAME2code
    
    df = ingestRules(rulesPath, rules[stepNum - 1], codes)
    
    print(df)
    
    toReclass, smaller, larger, largerNum = calculate(rasters[init] if currStep == 0
                                                      else rasters[reclassifiedPath], 
                                                      rasters[inter],
                                                      stepNum)
        
    reclassifiedPath = reclassify(df, toReclass, codes, largerNum, stepNum)
    
    return reclassifiedPath
```
# Execution
Execute functions in pattern
```python
step1 = stratify(ruleSteps)

arcpy.management.MakeRasterLayer(os.path.join(outputPath, 'reclassified1.tif'), 'reclassified1')

layers = mapView.listLayers()

rasters = {layer.name: Raster(arcpy.Describe(layer).catalogPath) 
           for layer in layers if layer.isRasterLayer}

step2 = stratify(ruleSteps, 1, 'reclassified1')
```
For systems get greater compute, alternative recursive function can be written.
