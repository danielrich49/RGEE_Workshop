# RGEE Workshop code

## What is Google Earth Engine?

[Google Earth Engine](https://earthengine.google.com/) is a cloud-based platform that allows users to have an easy access to a petabyte-scale archive of remote sensing data and run geospatial analysis on Google’s infrastructure. Currently, Google offers support only for Python and JavaScript. `rgee` will fill the gap **starting to provide support to R\!**. Below you will find the comparison between the syntax of `rgee` and the two Google-supported client libraries.

<table>
<tr>
<th> JS (Code Editor) </th>
<th> Python </th>
<th> R </th>
</tr>
<tr>
<td>
  
``` javascript
var db = 'CGIAR/SRTM90_V4'
var image = ee.Image(db)
print(image.bandNames())
#> 'elevation'
```

</td>
<td>

``` python
import ee
ee.Initialize()
db = 'CGIAR/SRTM90_V4'
image = ee.Image(db)
image.bandNames().getInfo()
#> [u'elevation']
```

</td>
<td>

``` r
library(rgee)
ee_Initialize()
db <- 'CGIAR/SRTM90_V4'
image <- ee$Image(db)
image$bandNames()$getInfo()
#> [1] "elevation"
```
</td>
</tr>
</table>

**Quite similar, isn’t it?**. However, there are additional smaller changes should consider when using Google Earth Engine with R. Please check the [consideration section](https://r-spatial.github.io/rgee/articles/considerations.html) before you start coding\!

## Installation

Install the `rgee` package from GitHub is quite simple, you just have to run in your R console as follows:

``` r
remotes::install_github("r-spatial/rgee")
```

**`rgee` depends on [sf](https://github.com/r-spatial/sf)**. Therefore, is necessary to install its external libraries, follow the installation steps specified [here](https://github.com/r-spatial/sf#installing). If you are using a Debian-based operating system, you probably need to install **virtualenv** as well.

```
sudo pip3 install virtualenv
```

#### Docker image
    
    docker pull csaybar/rgee
    docker run -d -p 8787:8787 -e USER=rgee -e PASSWORD=rgee --name rgee-dev csaybar/rgee

After that, in your preferred browser, run:

    127.0.0.1:8787

## setup

Prior to using `rgee` you will need to install a **Python version higher than 3.5** in their system. `rgee` counts with an installation function (ee_install) which helps to setup `rgee` correctly:

```r
library(rgee)

## It is necessary just once
ee_install()

# Initialize Earth Engine!
ee_Initialize()
```

Additionally, you might use the functions below for checking the status of rgee dependencies and delete credentials.

```r
ee_check() # Check non-R dependencies
ee_clean_credentials() # Remove credentials of a specific user
ee_clean_pyenv() # Remove reticulate system variables
```

Also, consider looking at the [setup section](https://r-spatial.github.io/rgee/articles/setup.html) for more information on customizing your Python installation.

## Package Conventions

  - All `rgee` functions have the prefix ee\_. Auto-completion is
    your friend :).
  - Full access to the Earth Engine API with the prefix
    [**ee$…**](https://developers.google.com/earth-engine/).
  - Authenticate and Initialize the Earth Engine R API with
    [**ee\_Initialize**](https://r-spatial.github.io/rgee/reference/ee_Initialize.html). It is necessary once by session!.
  - `rgee` is “pipe-friendly”, we re-exports %\>%, but `rgee` does not require its use.

## Quick Demo

### 1. Visualize a Landsat image collection in a true color composite 

```{r}
image <- ee$Image("LANDSAT/LC08/C01/T1/LC08_044034_20140318")
            
Map$centerObject(image)

vizParams <- list(
  bands = c("B5", "B4", "B3"),
  min = 5000, max = 15000, gamma = 1.3
)

Map$addLayer(image, vizParams, "Landsat 8 True color Composite")

```

<p align="center">
  <img src="https://raw.githubusercontent.com/MVanDenburg92/RGEE_Workshop/main/images/image_1.PNG" width=80%>
</p>

### 2.Visualize night-time lights

Load dataset and calcutalte average 
```{r}
dataset  <- ee$ImageCollection('NOAA/DMSP-OLS/NIGHTTIME_LIGHTS')$filter(ee$Filter$date('2010-01-01', '2010-12-31'))

nighttimeLights <-  dataset$select('stable_lights')

data_reduce <- nighttimeLights$reduce(ee$Reducer$mean())

```


```{r}
nighttimeLightsVis <-  list(
  min =  3.0,
  max =  60.0
)

Map$setCenter(7.82, 49.1, 4)

Map$addLayer(data_reduce, nighttimeLightsVis, 'Nighttime Lights')
```

<p align="center">
  <img src="https://raw.githubusercontent.com/MVanDenburg92/RGEE_Workshop/main/images/nighttime2.png" width=80%>
</p>

### 3. Compute the trend of night-time lights ([JS version](https://github.com/google/earthengine-api))

Authenticate and Initialize the Earth Engine R API.

``` r
library(rgee)
ee_Initialize()
#ee_reattach() # reattach ee as a reserve word
```

Adds a band containing image date as years since 1991.

``` r
createTimeBand <-function(img) {
  year <- ee$Date(img$get('system:time_start'))$get('year')$subtract(1991L)
  ee$Image(year)$byte()$addBands(img)
}
```

Map the time band creation helper over the [night-time lights collection](https://developers.google.com/earth-engine/datasets/catalog/NOAA_DMSP-OLS_NIGHTTIME_LIGHTS).

``` r
collection <- ee$
  ImageCollection('NOAA/DMSP-OLS/NIGHTTIME_LIGHTS')$
  select('stable_lights')$
  map(createTimeBand)
```

Compute a linear fit over the series of values at each pixel, visualizing the y-intercept in green, and positive/negative slopes as red/blue.

``` r
col_reduce <- collection$reduce(ee$Reducer$linearFit())
col_reduce <- col_reduce$addBands(
  col_reduce$select('scale'))
ee_print(col_reduce)
```

Create a interactive visualization\! 

``` r
Map$setCenter(9.08203, 47.39835, 3)
Map$addLayer(
  eeObject = col_reduce,
  visParams = list(
    bands = c("scale", "offset", "scale"),
    min = 0,
    max = c(0.18, 20, -0.18)
  ),
  name = "stable lights trend"
)

```

![rgee\_01](https://raw.githubusercontent.com/MVanDenburg92/RGEE_Workshop/main/images/nightime_rgee.png)


## Credits :bow:

Many many many thanks we would like to offer an **special thanks** :raised_hands: :clap: to Cesar Aybar
Author and Maintainer (https://github.com/r-spatial/rgee/) of **rgee**. It was his code  that created the demos used in today's workshop. In addition, the following third-party R/Python packages contributed indirectly to the develop of rgee:

  - **[gee\_asset\_manager - Lukasz Tracewski](https://github.com/tracek/gee_asset_manager)** 
  - **[geeup - Samapriya Roy](https://github.com/samapriya/geeup)**
  - **[geeadd - Samapriya Roy](https://github.com/samapriya/gee_asset_manager_addon)**  
  - **[cartoee - Kel Markert](https://github.com/KMarkert/cartoee)**
  - **[geetools - Rodrigo E. Principe](https://github.com/gee-community/gee_tools)**
  - **[landsat-extract-gee - Loïc Dutrieux](https://github.com/loicdtx/landsat-extract-gee)**
  - **[earthEngineGrabR - JesJehle](https://github.com/JesJehle/earthEngineGrabR)**
  - **[sf - Edzer Pebesma](https://github.com/r-spatial/sf)**
  - **[stars - Edzer Pebesma](https://github.com/r-spatial/stars)**
  - **[gdalcubes - Marius Appel](https://github.com/appelmar/gdalcubes)**

#### Readme template obtained from [dbparser](https://github.com/Dainanahan/dbparser)
