This is a more abstract and readable version of the logic used to stratify initial classification based various rules using raster operations.

```python
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
        (init * (10**int(math.ceil(math.log10(firstRasterSize)))) 
         + inter).save(reclassPath)
        
        return reclassPath, inter, init # smaller, larger
    else:
        (inter * (10**int(math.ceil(math.log10(secondRasterSize)))) 
         + init).save(reclassPath)
        
        return reclassPath, init, inter # smaller, larger
        
def reclassify(df, toReclass, codes, larger, step):
    remapValues = 0

    scalingFactor = 10**int(math.ceil(math.log10(larger)))

    remapValues = RemapValue(
        [[row[codes[0]] * scalingFactor + row[codes[1]], 
          row[codes[2]]] 
         for index, row in df.iterrows()]
    )
        
    Reclassify(os.path.join(outputPath, 'rasterCalc' + str(step) + '.tif'),
           "VALUE",
           RemapValue([[row[codes[0]] * 10 + row[codes[1]], 
                        row[codes[2]]] for index, row in df.iterrows()])).save(
               os.path.join(outputPath, 'reclassified' + str(step) + '.tif')
           )

def stratify(rules, currStep = 0):
    step, drop, init, inter, operation = rules[currStep].split('-')
    
    stepNum = int(step[-1]) # for example, get last character of string 'step1'

    codes = [init.upper() + 'code', #INIT_NAMEcode
             inter.upper() + 'code', #INTER_NAMEcode
             init.upper() + str(stepNum + 1) + 'code'] #INIT_NAME2Acode or INIT_NAME2code
    
    df = ingestRules(rulesPath, rules[stepNum - 1], codes)
    
    toReclass, smaller, larger = calculate(rasters[init], rasters[inter], stepNum)
        
    reclassify(df, toReclass, codes, larger, stepNum)
    
    if stepNum != len(rules):
        stratify(rules, stepNum)
```
