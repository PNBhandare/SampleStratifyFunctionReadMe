This is all the logic of the code mashed up into one function. For readability, they will be split in final implementation.

```python
def stratify(rules):
    step, drop, init, inter, operation = rules[0].split('-')
    
    df = ingestRules(rulesPath, rules[0], [init.upper + 'code', 
                                           inter.upper() 'code', 
                                           init.upper() + str(int(step[-1]) + 1) + 'code'])
    
    firstRasterSize = int(
        arcpy.GetRasterProperties_management(
            landfire, 
            "MAXIMUM").getOutput(0))
    
    secondRasterSize = int(
        arcpy.GetRasterProperties_management(
            unet, 
            "MAXIMUM").getOutput(0))
    
    if firstRasterSize > secondRasterSize:
        operation = init * (10**int(math.ceil(math.log10(firstRasterSize)))) + inter
    else:
        operation = inter * (10**int(math.ceil(math.log10(secondRasterSize)))) + init
        
    remapValues = 0

    if firstRasterSize > secondRasterSize:
        scalingFactor = 10**int(math.ceil(math.log10(firstRasterSize)))

        remapValues = RemapValue(
            [[row['UNETcode'] * scalingFactor + row['LANDFIREcode'], 
              row['UNET2Acode']] 
             for index, row in df.iterrows()]
        )
    else:
        scalingFactor = 10**int(math.ceil(math.log10(secondRasterSize)))

        remapValues = RemapValue(
            [[row['UNETcode'] * scalingFactor + row['LANDFIREcode'], 
              row['UNET2Acode']] 
             for index, row in df.iterrows()]
        )
        
    Reclassify(os.path.join(testPath, 'rasterCalc3Test.tif'),
           "VALUE",
           RemapValue([[row['UNET3code'] * 10 + row['LAKEcode'], row['UNET4code']] for index, row in df.iterrows()])).save(
               os.path.join(testPath, 'reclassified3Test.tif')
           )
    
    stratify(rules[step[-1]])
```
