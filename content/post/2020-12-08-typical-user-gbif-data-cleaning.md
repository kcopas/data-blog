---
title: Common things to look out for when post-processing GBIF downloads
author: John Waller
date: '2021-02-17'
slug: gbif-filtering-guide
categories:
  - GBIF
tags: []
lastmod: '2020-12-08T09:51:20+01:00'
draft: no
keywords: []
description: ''
authors: ''
comment: no
toc: ''
autoCollapseToc: no
postMetaInFooter: no
hiddenFromHomePage: no
contentCopyright: no
reward: no
mathjax: no
mathjaxEnableSingleDollar: no
mathjaxEnableAutoNumber: no
hideHeaderAndFooter: no
flowchartDiagrams:
  enable: no
  options: ''
sequenceDiagrams:
  enable: no
  options: ''
---

> Post was updated on April 20 2022 to accommodate changes to dwc:establishmentMeans vocabulary handling. 

Here I present a **checklist** for filtering **GBIF downloads**. 

In this guide, I will assume you are **familar with R**. This guide is also somewhat general, **so your solution might differ**. 
This guide is intended to give you a checklist of **common things** to look out for when post-processing GBIF downloads. 

<!--more-->

<!-- ## The checklist :  -->

<!-- - check **default geospatial issues** -->
<!-- - check **occurrenceStatus** -->
<!-- - check **basisOfRecord** ("FOSSIL_SPECIMEN" "LIVING_SPECIMEN")  -->
<!-- - check **establishmentMeans** ("MANAGED", "INTRODUCED", "INVASIVE", "NATURALISED") -->
<!-- - check **event date** or year -->
<!-- - check **uncertainty** (coordinatePrecision, coordinateUncertaintyInMeters) -->
<!-- - check if **zero coordinates** (decimalLatitude=0 or decimallongitude= 0) -->
<!-- - check if **country centroid** or other centroid -->
<!-- - check if in **ocean** -->
<!-- - check for **location duplicates**  -->

Below is an example a **filtering checklist script** that would work for most users. Individual users might want to **add/remove** some steps. After the script, I discuss each of these steps in **more detail below**. I would recommend looking at these two articles before continuing:

1. [Getting GBIF mediated occurrence data with rgbif](https://docs.ropensci.org/rgbif/articles/getting_occurrence_data.html)
2. [Setting up GBIF credentials](https://docs.ropensci.org/rgbif/articles/gbif_credentials.html)

```R 
library(rgbif)
library(dplyr)
library(CoordinateCleaner)

taxonkey <- name_backbone("Calopteryx xanthostoma")$usageKey

# set up gbif credentials first
# https://docs.ropensci.org/rgbif/articles/gbif_credentials.html

gbif_download <- occ_download(
  pred("taxonKey", taxonkey),
  pred("hasCoordinate", TRUE), 
  pred("hasGeospatialIssue", FALSE), # remove GBIF default geospatial issues
  format = "SIMPLE_CSV") 

occ_download_wait(gbif_download) 

# filtering pipeline  
gbif_download %>%
  occ_download_get() %>%
  occ_download_import() %>%
  setNames(tolower(names(.))) %>% # set lowercase column names to work with CoordinateCleaner
  filter(occurrencestatus  == "PRESENT") %>%
  filter(!basisofrecord %in% c("FOSSIL_SPECIMEN","LIVING_SPECIMEN")) %>%
  filter(year >= 1900) %>% 
  filter(coordinateprecision < 0.01 | is.na(coordinateprecision)) %>% 
  filter(coordinateuncertaintyinmeters < 10000 | is.na(coordinateuncertaintyinmeters)) %>%
  filter(!coordinateuncertaintyinmeters %in% c(301,3036,999,9999)) %>% 
  filter(!decimallatitude == 0 | !decimallongitude == 0) %>%
  cc_cen(buffer = 2000) %>% # remove country centroids within 2km 
  cc_cap(buffer = 2000) %>% # remove capitals centroids within 2km
  cc_inst(buffer = 2000) %>% # remove zoo and herbaria within 2km 
  cc_sea() %>% # remove from ocean 
  distinct(decimallongitude,decimallatitude,specieskey,datasetkey, .keep_all = TRUE) %>%
  glimpse() # look at results of pipeline
```

<!--
It is usually a good idea to move as much of your data filtering into the **download stage** as possible. Some filters I list above are **not available** at the download stage.

Here a script that moves **most** filters from above into the **downloads api**. 

```R 
library(rgbif)

user="" # GBIF user name
pwd="" # GBIF password
email="" # your email

gbif_download = occ_download(
type="and",
pred("taxonKey", 5052020),
pred("hasGeospatialIssue", FALSE),
pred("hasCoordinate", TRUE),
pred_gte("year", 1900),
pred_not(pred("basisOfRecord", "FOSSIL_SPECIMEN")),
pred_or(
pred_not(pred("establishmentMeans","MANAGED")),
pred_not(pred_notnull("establishmentMeans"))
),
pred_or(
pred_not(pred("establishmentMeans","INTRODUCED")),
pred_not(pred_notnull("establishmentMeans"))
),
pred_or(
pred_not(pred("establishmentMeans","INVASIVE")),
pred_not(pred_notnull("establishmentMeans"))
),
pred_or(
pred_not(pred("establishmentMeans","NATURALISED")),
pred_not(pred_notnull("establishmentMeans"))
),
pred_or(
pred_lt("coordinateUncertaintyInMeters",10000),
pred_not(pred_notnull("coordinateUncertaintyInMeters"))
),
format = "SIMPLE_CSV",
user=user,pwd=pwd,email=email
)

# rest of the pipeline after download
# not yet possible to use these filters in the download stage 

gbif_download %>%
filter(coordinateprecision < 0.01 | is.na(coordinateprecision)) %>% 
filter(!coordinateuncertaintyinmeters %in% c(301,3036,999,9999)) %>% 
filter(!decimallatitude == 0 | !decimallongitude == 0) %>%
cc_cen(buffer = 2000) %>% # remove country centroids within 2km 
cc_cap(buffer = 2000) %>% # remove capitals centroids within 2km
cc_inst(buffer = 2000) %>% # remove zoo and herbaria within 2km 
cc_sea() %>% # remove from ocean 
distinct(decimallongitude,decimallatitude,specieskey,datasetkey, .keep_all = TRUE) %>%
glimpse() # look at results of pipeline

```
-->

## GBIF default geospatial issues 

GBIF removes **common geospatial issues** by default if you choose to have data with a location. 

The following things will be removed: 

1. **Zero coordinate** : Coordinates are exactly (0,0). [null island](https://en.wikipedia.org/wiki/Null_Island)
2. **Country coordinate mismatch** : The coordinates fall outside of the given country's polygon.
2. **Coordinate invalid** : GBIF is unable to interpret the coordiantes. 
2. **Coordinate out of range** : The coordinates are outside of the range for decimal lat/lon values ((-90,90), (-180,180)).

You can do this on the [web portal](https://www.gbif.org/occurrence/search?has_coordinate=true&has_geospatial_issue=false&occurrence_status=present). 

## Absence data

GBIF now has a field called **occurrence status**, which will tell you whether an occurrence is a presence or absence. 

```R
gbif_download %>%
filter(occurrenceStatus  == "PRESENT")
```
You can also do this on the [web portal](https://www.gbif.org/occurrence/search?occurrence_status=present&q=) before downloading. 


## Fossils and living specimens

You might want to remove **fossils** and **living specimens**, and **non-naturally established species**.

```R
gbif_download %>% 
filter(!basisOfRecord %in% c("FOSSIL_SPECIMEN","LIVING_SPECIMEN")) %>%
```

## Old records

You might also want to remove old records. 

```R 
gbif_download %>% 
filter(year >= 1900)
``` 

You can also do this on the [web portal](https://www.gbif.org/occurrence/search?year=1900,2020&occurrence_status=present) before downloading.

## High uncertainty

There are two fields that come with **simple csv downloads** that give uncertainty. 

1. **coordinatePrecision** : A decimal representation of the precision of the coordinates. 
2. **coordinateUncertaintyInMeters** : the uncertainty of the coordinates in meters. 

These fields are not frequently used by publishers (around 600M occurrences do not fill in either), **so filter with caution**. I have kept missing values in my example. 

If you want to be sure that a point has the acceptable level of uncertainty or precision for your study, you can remove those with missing values, but this will be removing a lot "ok" records.  

```R
# I keep missing values here
gbif_download %>% 
filter(
coordinatePrecision < 0.01 | is.na(coordinatePrecision)
) %>% 
filter(
coordinateUncertaintyInMeters < 1000 | is.na(coordinateUncertaintyInMeters)
) 

```

You also want to remove records with [known default values](https://github.com/ropensci/CoordinateCleaner/issues/55) for **coordinateUncertaintyInMeters**. These can be **GeoLocate centroids** or some other default. It is good to remove them because usually the uncertainy is larger than what is stated by the value. 

```R
gbif_download %>% 
filter(!coordinateUncertaintyInMeters %in% c(301,3036,999,9999)) 
# known inaccurate default values

```
## Points along equator or prime merdidian 

Point plotted along the **prime meridian** or **equator**.

As of the writing of this guide there are 37K occurrences along the **equator** and 28K along the **Prime meridian**. 

```R 
gbif_download %>% 
filter(decimalLatitude == 0 | decimalLongitude == 0)

# see also CoordinateCleaner::cc_zero()
```

## Country centroids

You can remove country centroids and province centroids using **CoordinateCleaner**.

```R 
library(CoordinateCleaner)

gbif_download %>% 
cc_cen(
lon = "decimalLongitude", 
lat = "decimalLatitude", 
buffer = 2000, # radius of circle around centroid to look for centroids
value = "clean",
test="both")  
```
Centroids tend to be more common for geocoded museum collections (PRESERVED_SPECIMEN), so you might want to only filter centroids for preserved specimens, since other occurrences might be false positives.  

You might also want to filter **country capitals**. 

```R 
gbif_download %>% 
cc_cap(
lon = "decimalLongitude", 
lat = "decimalLatitude", 
buffer = 2000, # radius of circle around centroid to look for centroids
value = "clean")

```

## Zoo and herbaria locations

Publishers do not always fill in the establishmentMeans="MANAGED" or basisOfRecord="LIVING_SPECIMEN", so it is usually good to filter known **zoo** and **herbaria** locations. 

```R 
library(CoordinateCleaner)

gbif_download %>% 
cc_inst(
lon = "decimalLongitude",
lat = "decimalLatitude",
buffer = 2000,
value = "clean",
verbose = TRUE
)

```

## In the ocean

Obviously not to be used with marine species. If marine, you might want to do the opposite. 

```R 
library(CoordinateCleaner)

gbif_download %>% 
cc_sea(
lon = "decimalLongitude",
lat = "decimalLatitude"
)

```

## Location duplicates

Some of you will want to remove potential **location duplicates**. 

```R 
gbif_download %>% 
distinct(decimalLongitude,decimalLatitude,speciesKey,datasetKey, .keep_all = TRUE) 
```

It is probably a good idea to keep the **datasetKey** for citing the download later, if you post a derived dataset on Zenodo or something similar. 

In general removing duplicates is not difficult based on location. GBIF also has a experimental feature for indentifying [related records](https://www.gbif.org/news/4U1dz8LygQvqIywiRIRpAU/new-data-clustering-feature-aims-to-improve-data-quality-and-reveal-cross-dataset-connections). It is, however, not optimized for data filtering yet. 

## R packages for filtering data

There are currently 3 **R packages** for filtering GBIF occurrences:

1. [CoordinateCleaner](https://github.com/ropensci/CoordinateCleaner)
2. [biogeo](https://cran.r-project.org/web/packages/biogeo/index.html)
3. [scrubr](https://github.com/ropensci/scrubr)

**CoordinateCleaner** is probably the most [most complete](https://cran.r-project.org/web/packages/CoordinateCleaner/vignettes/Comparison_other_software.html
).

## Additional filters

There are some **additional** things you might want to check for which invovle more judgement calls: 

* outliers  
* metagenomics 
* outside native ranges
* gridded datasets
* automated identifications

I have written some companion R scripts for handling these issues as well [link to scripts](https://github.com/jhnwllr/GBIF_additional_filters/blob/main/README.md).




