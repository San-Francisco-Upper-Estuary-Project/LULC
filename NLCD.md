---
title: "USGS NLCD Tabulation and Summary"
author: "Skyler Elmstrom"
date: "3/7/2021"
output:
  html_document:
    theme: lumen
    code_download: true
    keep_md: true
---



<br>

<details>
  <summary><b>Required Libraries</b></summary>

```r
# devtools::install_github("WWU-IETC-R-Collab/IETC")
library(IETC)

# devtools::install_github("ropensci/FedData") # Use Fed Data Version 3.0.0 or higher. CRAN has 2.5.7
library(FedData)

library(rgdal) # R geoprocessing tools
library(raster) # Raster data manipulation
library(sf) # vector data manipulation
library(tidyverse)
```
</details>
<br>

<details>
  <summary><b>Functions and Class Key</b></summary>

```r
# Function to Tabulate raster::extract() Output by Polygon
# http://zevross.com/blog/2015/03/30/map-and-analyze-raster-data-in-r/
tabFunc <- function(indx, extracted, region, regname) {
  dat<-as.data.frame(table(extracted[[indx]]))
  dat$name<-region[[regname]][[indx]]
  return(dat)
}

# Function for converting pixel count to square kilometers
pc2sqkm <- function(x, in_raster) {
  apc <- prod(raster::res(in_raster))
  x * apc / 1e+6
}

# Create NLCD Class Key
# Based on: https://www.mrlc.gov/data/legends/national-land-cover-database-2016-nlcd2016-legend
NLCD.class <- c('OpenWater' = '11',
                'PerennialIceSnow' = '12',
                'DevOpenSpace' = '21',
                'DevLowInt' = '22',
                'DevMedInt' = '23',
                'DevHighInt' = '24',
                'BarrenLand' = '31',
                'DeciduousForest' = '41',
                'EvergreenForest' = '42',
                'DwarfShrub' = '51',
                'ShrubScrub' = '52',
                'GrassHerb' = '71',
                'SedgeHerb' = '72',
                'Lichens' = '73',
                'Moss' = '74',
                'PastureHay' = '81',
                'CultivatedCrops' = '82',
                'WoodyWetlands' = '90',
                'EmergentHerbWetlands' = '95'
                )
```
</details>
<br>

## Tabulating NLCD by Risk Region in R


```r
# Load Risk Regions
USFE.riskregions.z <- "https://github.com/WWU-IETC-R-Collab/CEDEN-mod/raw/main/Data/USFE_RiskRegions_9292020.zip"
USFE.riskregions <- IETC::unzipShape(USFE.riskregions.z) # loads GitHub zipped shapefile as an sf object

# Alternative NLCD data at mrlc.gov/viewer using the following extent
# Latitude dd: 36.75543, 38.92584
# Longitude dd: -123.72037, -120.52298
# https://www.mrlc.gov/viewer/?downloadBbox=36.75543,38.92584,-123.72037,-120.52298

# Get the NLCD (USA ONLY)
# Returns a raster
USFE.NLCD <- get_nlcd(template = USFE.riskregions,
                 year = 2016,
                 dataset = "Land_Cover",
                 label = "USFE")

# Transform Risk Regions to match raster data
USFE.riskregions.NLCD <- as_Spatial(st_zm(USFE.riskregions)) %>%
  spTransform(crs(USFE.NLCD))

# Mask Study Area
USFE.NLCD.mask <- raster::mask(USFE.NLCD, USFE.riskregions.NLCD)

# Check Plot with raster::plot
# raster::plot(USFE.NLCD.mask)

# Extract Raster Values
USFE.LULC <- raster::extract(USFE.NLCD.mask, USFE.riskregions.NLCD) # be patient, this takes a while

# Tabulate raster::extract() lists
# TO-DO: Modify table to be long format by class and risk region
USFE.LULC.tabulated <- lapply(seq(USFE.LULC), tabFunc, USFE.LULC, USFE.riskregions.NLCD, "Subregion") %>% # tabulate result lists from raster::extract
  do.call("rbind", .) %>% # combine the tabulated raster::extract
  pivot_wider(names_from = Var1, # pivot classes wide
              values_from = Freq) %>%
  dplyr::select(name, any_of(NLCD.class)) %>% # replace column class ID # with class name
  mutate(across(where(is.numeric), ~pc2sqkm(., USFE.NLCD.mask))) # convert pixel counts to sq km

# Write Final Table to Outputs
write_csv(USFE.LULC.tabulated, "Output/NLCD_LULC.csv")
```
