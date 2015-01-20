---
title: "Quantitative Disk Assay"
author: "Aleeza C. Gerstein"
date: "2015-01-20"
output: rmarkdown::html_vignette
output:
	html_document:
		fig_width: 2
		fig_height: 2
vignette: >
  %\VignetteIndexEntry{Quantitative Disk Assay}
  %\VignetteEngine{knitr::rmarkdown}
  \usepackage[utf8]{inputenc}
---

# Introduction to diskImageR

### January 19, 2015
---

diskImageR provides a quantitative way to analyze photographs taken from disk diffusion assays. Specifically, 

<center>
<img src="../inst/pictures/p2_30_a.JPG"  style="width: 40%; height: 40%" style="float:left," alt="" /> 
</center>

## Prepare plates and photographs
The analysis done by diskImageR will only be as good as your disk assay plates and the photographs you take. Plates should always be labelled on the side, not on the bottom. Care should be taken when setting up the camera to take photographs, as you want the lighting conditions to be as uniform as possible, without any shadows on the plates. Camera settings should be manual rather than automatic as much as possible. Once you have the set of photographs that you want to be analyzed together they should be placed in the same directory, with nothing else inside that directory.

Photograph naming can be used downstream to have diskImageR do a number of statistical things (e.g., averaging across replicates, caulculations of variance, t-tests). The general format is "strain_factor1_factor2_rep.pdf". Conversely, if you intend to do all the statistical analysis later, photographs can be named anything, even numbered.

Finally, photographs should be cropped carefully around the disk.

<b> Important! </b> There can not be any spaces or special characters in any of the folder names that are in the path that will lead to your pictures or the directory that you will use as the main project folder (i.e., the place where all the output files from this package will go). 

## Run the imageJ macro on the set of photographs
The first step in the diskImageR pipeline is to run the imageJ macro on the photograph directory. 

<b> Important! </b> imageJ must be installed on your computer. ImageJ is a free, public domain Java image proessing program available for download <a href="http://rsb.info.nih.gov/ij/download.html"> here</a>. Take note of the path to imageJ, as this will be needed for the first function.

From each photograph, the macro (in imageJ) will automatically determine where the disk is located on the plate, find the center of the disk, and draw 40mm lines out from the center of the disk every 5 degrees. For each line, the pixel intensity will be determined at many points along the line. This data will be stored in the folder "imageJ-out" on your computer, with one file for each photograph.

This step can be completed using one of two functions.
```r
#This function allows you to run the imageJ macro through a user-interface with pop-up boxes to select 
#where you want the main project directory to be and where to find the location of the photograph directory.
runIJ("projectName")
```

```r
#If you are more comfortable with R, and don't want to be bothered with pop-up boxes, use the 
#alternate function to supply the desired main project directory and photograph directory locations.
runIJManual("vignette", projectDir= getwd(), pictureDir = file.path(.libPaths(), "diskImageR", "pictures", ""), imageJLoc = "loc2")
```

Depending on where imageJ is located, the script may not run unless you specify the filepath. See ?runIJ for more details.

If you want to access the output of these functions in a later R session you can with
```r
readInExistingIJ("projectName") 	#can be any project name, does not have to be the same as previously used
```

### [optional] Plot the imageJ output
To plot pixel intensity from the average from all photographs use

```r
plotRaw("vignette", savePDF=FALSE)
```

![plot of chunk unnamed-chunk-1](figure/unnamed-chunk-1-1.png) 

## Run the maximum likelihood analysis 
The next step is to use maximum likelihood to find the logistic and double logistic equations that best describe the shape of the imageJ output data. These data follow a characteristic "S-shape" curve, so the standard logistic equation is used where asym is the asymptote, od50 is the midpoint, and scal is the slope at od50 divided by asym/4.
$$
y = \frac{asym*exp(scal(x-od50))}{1+exp(scal(x-od50))}+N(0, \sigma)
$$

We often observed disk assays that deviated from the single logistic, either rising more linearly than expected at low cell density, or with an intermediate asymptote around the midpoint. To fascilitate fitting these curves, we fit a double logistic, which allows greater flexibility. Our primary goal in curve fitting is to capture an underlying equation that fits the observed data, rather than to test what model fits better.
$$
y = \frac{asymA*exp(scalA(x-od50A))}{1+exp(scalA(x-od50A))}+\frac{asymB*exp(scalB(x-od50B))}{1+exp(scalB(x-od50B))}+N(0, \sigma)
$$

From these functions we substract off the plate background intensity from all values; this is common across all pictures taken at the same time and is determined from the observed pixel intensity on a plate with a clear halo (specified by the user). We then use the parameters identified in the logistic equations to determine the resistance parameters.

* <b>Resistance</b>
	: asymA+asymB are added together to determine the maximum level of intensity (= cell density) achieved on each plate. The level of resistance (zone of inhibition, ZOI), is calculated by asking what x value (distance in mm) corresponds to the point where 80%, 50% and 20% reduction in growth occurs (corresponding to *ZOI80*, *ZOI50*, and *ZOI20*)
* <b>Tolerance</b>
	: the 'rollmean' function from the zoo package is used to calculate the area under the curve (AUC)  in slice from the disk edge to each ZOI cutoff. This achieved growth is then compared to the potential growth, namely, the area of a rectangle with length and height equal to the ZOI. The calculated paramaters are thus the fraction of full growth in this region (*fAUC80*, *fACU50*, *fAUC20*).
* <b>Sensitivity</b>
	: the ten data points on either side of the midpoint (od50) from the single logistic equation are used to find the slope of the best fit linear model using the lm function in R.


```r
maxLik("vignette", clearHalo=1, savePDF=FALSE, ZOI="all", needML=FALSE)
```

```
## 
## Using existing ML results vignette.ML & vignette.ML2
```

![plot of chunk unnamed-chunk-2](figure/unnamed-chunk-2-1.png) 

### [OPTIONAL] Save the maximum likelihood results
It is possible to save the maximum likelihood results using
```
saveMLParam("vignette")
```
This will save a .csv file into the *paramter_files* directory that contains parameter estimates for asym, od50, scal and sigma, as well as the log likelihood of the single and double logistic models
 
## Output and save the results 
The last step is to save a dataframe with the resistance parameter estimates. 


```r
createDataframe("vignette", clearHalo = 1)
```

```
## 
## vignette.df has been written to the global environment
## Saving file: /Users/acgerstein/Documents/Postdoc/Research/diskImageR/vignettes/parameter_files/vignette/vignette_df.csv
## vignette_df.csv can be opened in MS Excel.
```

```r
vignette.df
```

```
##      name lines type ZOI80 ZOI50 ZOI20 fAUC80 fAUC50 fAUC20 slope
## 1 p1_30_a    p1   30    11    14    17   0.37   0.26   0.29 133.2
## 2 p2_30_a    p2   30     0    13    17   1.00   0.94   0.63  75.6
```