This is a more abstract and readable version of the logic used to stratify initial classification based various rules using raster operations. Headers and descriptors and additional documentation will be added.

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
    
    print('Got size')
    
    if firstRasterSize > secondRasterSize:
        print('Trying math')
        (inter * (10**int(math.ceil(math.log10(firstRasterSize)))) 
         + init).save(reclassPath)
        print('Mathed')
        
        return reclassPath, inter, init, firstRasterSize # smaller, larger
    else:
        print('Trying math')
        (init * (10**int(math.ceil(math.log10(secondRasterSize)))) 
         + inter).save(reclassPath)
        print('Mathed')
        
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
    
    print(larger)
        
    reclassifiedPath = reclassify(df, toReclass, codes, largerNum, stepNum)
    
    print(reclassifiedPath)
    
    return reclassifiedPath
```
