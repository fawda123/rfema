rfema (R FEMA)
================

-   [Introduction](#introduction)
-   [Why rfema?](#why-rfema)
-   [Installation](#installation)
-   [Usage](#usage)

[![R-CMD-check](https://github.com/dylan-turner25/rfema/workflows/R-CMD-check/badge.svg)](https://github.com/dylan-turner25/rfema/actions)
[![Project Status: Active – The project has reached a stable, usable
state and is being actively
developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)

[![Lifecycle:
stable](https://img.shields.io/badge/lifecycle-stable-brightgreen.svg)](https://www.tidyverse.org/lifecycle/#stable)
[![Codecov test
coverage](https://codecov.io/gh/dylan-turner25/rfema/branch/main/graph/badge.svg)](https://codecov.io/gh/dylan-turner25/rfema?branch=main)
[![Status at rOpenSci Software Peer
Review](https://badges.ropensci.org/484_status.svg)](https://github.com/ropensci/software-review/issues/484)

<!-- badges: start -->

<!-- [![R-CMD-check](https://github.com/dylan-turner25/rfema/workflows/R-CMD-check/badge.svg)](https://github.com/dylan-turner25/rfema/actions) -->
<!-- badges: end -->

## Introduction

`rfema` allows users to access The Federal Emergency Management Agency’s
(FEMA) publicly available data through the open FEMA API. The package
provides a set of functions to easily navigate and access all data sets
provided by FEMA, including (but not limited to) data from the National
Flood Insurance Program and FEMA’s various disaster aid programs.

FEMA data is publicly available at the open [FEMA
website](https://www.fema.gov/about/openfema/data-sets) and is available
for bulk download, however, the files are sometimes very large (multiple
gigabytes) and many times users do not need all records for a data
series (for example: many users may only want records for a single state
for several years). Using FEMA’s API is a good option to circumvent
working with the bulk data files, but can be inaccessible for those
without prior API experience. This package contains a set of functions
that allows users to easily identify and retrieve data from FEMA’s API
without needing any technical knowledge of APIs. Notably, the FEMA API
does not require an API key meaning the package is extremely accessible
regardless of if the user has ever interacted with an API.

The rest of this page explains the benefits of the package and
demonstrates basic usage of the package. For those looking for more in
depth examples of how to use the package in your workflow, consider
reading the [Getting
Started](https://github.com/dylan-turner25/rfema/blob/main/vignettes/getting_started.md)
vignette.

In accordance with the Open Fema terms and conditions: This product uses
the Federal Emergency Management Agency’s Open FEMA API, but is not
endorsed by FEMA. The Federal Government or FEMA cannot vouch for the
data or analyses derived from these data after the data have been
retrieved from the Agency’s website(s). Guidance on FEMA’s preferred
citation for Open FEMA data can be found at:
<https://www.fema.gov/about/openfema/terms-conditions>.

## Why rfema?

What are the advantages of accessing the FEMA API through the `rfema`
package as compared to accessing the API directly? In short, the `rfema`
package handles much of the grunt work associated with constructing API
queries, dealing with API limits, and applying filters or other
parameters. Suppose one wants to obtain data on all of the flood
insurance claims in Broward County, FL between 2010 and 2012. The
following code obtains that data without the use of the `rfema` package.
As can be seen it requires quite a few lines of code, in part due to the
API limiting calls to 1000 records per call which can make obtaining a
full data set cumbersome.

``` r
# Code needed to obtain data on flood insurance claims in Broward County, FL without the rfema package ------------------

# define the url for the appropriate api end point
base_url <- "https://www.fema.gov/api/open/v1/FimaNfipClaims"


# append the base_url to apply filters
filters <- "?$inlinecount=allpages&$top=1000&$filter=(countyCode%20eq%20'12011')%20and%20(yearOfLoss%20ge%20'2010')%20and%20(yearOfLoss%20le%20'2012')"

api_query <- paste0(base_url, filters)

# run a query setting the top_n parameter to 1 to check how many records match the filters
record_check_query <- "https://www.fema.gov/api/open/v1/FimaNfipClaims?$inlinecount=allpages&$top=1&$select=id&$filter=(countyCode%20eq%20'12011')%20and%20(yearOfLoss%20ge%20'2010')%20and%20(yearOfLoss%20le%20'2012')"

# run the api call and determine the number of matching records
result <- httr::GET(record_check_query)
jsonData <- httr::content(result)        
n_records <- jsonData$metadata$count 


# calculate number of calls neccesary to get all records using the 
# 1000 records/ call max limit defined by FEMA
iterations <- ceiling(n_records / 1000)

# initialize a skip counter which will indicate where in the full 
# data set each API call needs to start from.
skip <- 0

# make however many API calls are neccesary to get the full data set
for (i in seq(from = 1, to = iterations, by = 1)) {
  # As above, if you have filters, specific fields, or are sorting, add
  # that to the base URL or make sure it gets concatenated here.
  result <- httr::GET(paste0(api_query, "&$skip=", (i - 1) * 1000))
  if (result$status_code != 200) {
    status <- httr::http_status(result)
    stop(status$message)
  }
  json_data <- httr::content(result)[[2]]
  
  # for data returned as a list of lists, correct any discrepancies
  # in the length of the lists by adding NA values to the shorter lists
  
  # calculate longest list
  max_list_length <- max(sapply(json_data, length))
  
  # add NA values to lists shorter than the max list length
  json_data <- lapply(json_data, function(x) {
    c(x, rep(NA, max_list_length - length(x)))
  })
  
  if (i == 1) {
    # bind the data into a single data frame
    data <- data.frame(do.call(rbind, json_data))
  } else {
    data <- dplyr::bind_rows(
      data,
      data.frame(do.call(rbind, json_data))
    )
  }
}

 
# remove the html line breaks from returned data frame (if there are any)  
data <- as_tibble(lapply(data, function(data) gsub("\n", "", data)))

# view the retrieved data
data
```

    ## # A tibble: 2,119 × 40
    ##    agricultureStruct… asOfDate  baseFloodElevat… basementEnclosur… reportedCity 
    ##    <chr>              <chr>     <chr>            <chr>             <chr>        
    ##  1 FALSE              2021-07-… 8                NULL              Temporarily …
    ##  2 FALSE              2021-09-… 6                0                 Temporarily …
    ##  3 FALSE              2021-09-… 4                0                 Temporarily …
    ##  4 FALSE              2021-09-… 6                0                 Temporarily …
    ##  5 FALSE              2021-09-… NULL             NULL              Temporarily …
    ##  6 FALSE              2021-09-… 6                NULL              Temporarily …
    ##  7 FALSE              2021-07-… NULL             NULL              Temporarily …
    ##  8 FALSE              2021-07-… 7                NULL              Temporarily …
    ##  9 FALSE              2021-07-… 7                NULL              Temporarily …
    ## 10 FALSE              2021-09-… NULL             0                 Temporarily …
    ## # … with 2,109 more rows, and 35 more variables: condominiumIndicator <chr>,
    ## #   policyCount <chr>, countyCode <chr>, communityRatingSystemDiscount <chr>,
    ## #   dateOfLoss <chr>, elevatedBuildingIndicator <chr>,
    ## #   elevationCertificateIndicator <chr>, elevationDifference <chr>,
    ## #   censusTract <chr>, floodZone <chr>, houseWorship <chr>, latitude <chr>,
    ## #   longitude <chr>, locationOfContents <chr>, lowestAdjacentGrade <chr>,
    ## #   lowestFloorElevation <chr>, numberOfFloorsInTheInsuredBuilding <chr>, …

Compare the above block of code to the following code which obtains the
same data using the `rfema` package. The `rfema` package allows the same
request to be made with two lines of code. Notably, the `open_fema()`
function handles checking the number of records and implements an
iterative loop to deal with the 1000 records/call limit.

``` r
# define a list of filters to apply
filterList <- list(countyCode = "= 12011",yearOfLoss = ">= 2010", yearOfLoss = "<= 2012")


# Make the API call using the `open_fema` function. The function will output a 
# status message to the console letting you monitor the progress of the data retrieval.
data <- rfema::open_fema(data_set = "fimaNfipClaims",ask_before_call = F, filters = filterList)

# view data
data
```

    ## # A tibble: 2,119 x 40
    ##    agricultureStructur~ asOfDate            baseFloodElevati~ basementEnclosure~
    ##    <chr>                <dttm>              <chr>             <chr>             
    ##  1 FALSE                2021-07-25 00:00:00 8                 NULL              
    ##  2 FALSE                2021-11-21 00:00:00 NULL              NULL              
    ##  3 FALSE                2021-11-21 00:00:00 7                 NULL              
    ##  4 FALSE                2021-09-02 00:00:00 NULL              NULL              
    ##  5 FALSE                2021-09-02 00:00:00 6                 0                 
    ##  6 FALSE                2021-11-21 00:00:00 8                 NULL              
    ##  7 FALSE                2021-09-02 00:00:00 6                 NULL              
    ##  8 FALSE                2021-11-21 00:00:00 11                NULL              
    ##  9 FALSE                2021-07-04 00:00:00 NULL              NULL              
    ## 10 FALSE                2021-07-04 00:00:00 7                 NULL              
    ## # ... with 2,109 more rows, and 36 more variables: reportedCity <chr>,
    ## #   condominiumIndicator <chr>, policyCount <chr>, countyCode <chr>,
    ## #   communityRatingSystemDiscount <chr>, dateOfLoss <dttm>,
    ## #   elevatedBuildingIndicator <chr>, elevationCertificateIndicator <chr>,
    ## #   elevationDifference <chr>, censusTract <chr>, floodZone <chr>,
    ## #   houseWorship <chr>, latitude <chr>, longitude <chr>,
    ## #   locationOfContents <chr>, lowestAdjacentGrade <chr>, ...

The `rfema` package also returns data, where possible, in formats that
are easier to work with. For example, all functions return data as a
tibble with all date columns converted to POSIX format. This makes
plotting time series easy as the API call can be piped directly into a
`ggplot` plot. For example, the following is a plot of the number of
FEMA disaster declarations in response to hurricanes since 2010,
separated out by Florida versus the rest of the United States. In an
application where the most up to date data is required, this block of
code can be rerun to plot the most up to date data from the FEMA API.

``` r
library(ggplot2)
open_fema("DisasterDeclarationsSummaries", 
                  filters = list(declarationDate = ">= 2010-01-01",
                                 incidentType = "Hurricane"), 
                  ask_before_call = F) %>% 
  mutate(date = lubridate::floor_date(declarationDate,"year"),count = 1,
         Florida = factor(state == "FL")) %>%
  select(date,count,Florida) %>%
  group_by(date,Florida) %>%
  summarise(count = sum(count)) %>%
  ggplot(., aes(fill=Florida, y=count, x=date)) + 
    geom_bar(position="stack", stat="identity") +
  scale_fill_manual(name = "",values = c("grey80","forestgreen"),drop = FALSE,
                    labels = c("Rest of U.S.","Florida")) +
  labs(x = "year",y = "",
       title = "County Level FEMA Disaster Declarations for Hurricanes") +
  theme_light()
```

![](README_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

## Installation

Right now, the best way to install and use the `rfema` package is by
installing directly from GitHub using
`remotes::install_github("dylan-turner25/rfema")`. The FEMA API does not
require an API key, meaning no further steps need be taken to start
using the package.

## Usage

For those unfamiliar with the data sets available through the FEMA API,
a good starting place is to visit the [FEMA API documentation
page](https://www.fema.gov/about/openfema/data-sets). However, if you
are already familiar with the data and want to quickly reference the
data set names or another piece of meta data, using the
`fema_data_sets()` function to obtain a tibble of available data sets
along with associated meta data is a convenient option.

``` r
# store meta data for the available data sets as an object in the R environment
data_sets <- fema_data_sets()

# view the just retrieved data
data_sets
```

    ## # A tibble: 37 × 64
    ##    identifier  name    title   description   distribution.acce… distribution.fo…
    ##    <chr>       <chr>   <chr>   <chr>         <chr>              <chr>           
    ##  1 openfema-1  Public… Public… FEMA provide… https://www.fema.… csv             
    ##  2 openfema-1  Public… Public… FEMA provide… https://www.fema.… csv             
    ##  3 openfema-26 FemaWe… FEMA W… This data se… https://www.fema.… csv             
    ##  4 openfema-28 Hazard… Hazard… This dataset… https://www.fema.… csv             
    ##  5 openfema-36 FemaRe… FEMA R… Provides the… https://www.fema.… csv             
    ##  6 openfema-22 Emerge… Emerge… This dataset… https://www.fema.… csv             
    ##  7 openfema-45 Hazard… Hazard… The dataset … https://www.fema.… csv             
    ##  8 openfema-12 Housin… Housin… This dataset… https://www.fema.… csv             
    ##  9 openfema-8  DataSe… OpenFE… Metadata for… https://www.fema.… csv             
    ## 10 openfema-37 Hazard… Hazard… This dataset… https://www.fema.… csv             
    ## # … with 27 more rows, and 58 more variables: distribution.datasetSize <chr>,
    ## #   distribution.accessURL.1 <chr>, distribution.format.1 <chr>,
    ## #   distribution.datasetSize.1 <chr>, distribution.accessURL.2 <chr>,
    ## #   distribution.format.2 <chr>, distribution.datasetSize.2 <chr>,
    ## #   webService <chr>, dataDictionary <chr>, keyword <chr>, modified <chr>,
    ## #   publisher <chr>, contactPoint <chr>, mbox <chr>, accessLevel <chr>,
    ## #   landingPage <chr>, temporal <chr>, api <chr>, version <chr>, …

Once you have the name of the data set you want, simply pass it as an
argument to the `open_fema()` function which will return the data set as
a tibble. By default, `open_fema()` will warn you if the number of
records is greater than 1000 and present an estimated time required to
complete the records request. As the user, you will the be asked to
confirm that you want to retrieve all of the available records (for many
data sets the total records is quite large). To turn off this feature,
set the parameter `ask_before_call` equal to FALSE. To limit the number
of records returned, specify the `top_n` argument. This is useful for
exploring a data set without retrieving all records.

``` r
# obtain the first 10 records from the fimaNfipClaims data set.
# Note: the data_set argument is not case sensative
retrieved_data <- open_fema(data_set = "fimanfipclaims", top_n = 10)

# view the data
retrieved_data
```

    ## # A tibble: 10 × 40
    ##    agricultureStructur… asOfDate            baseFloodElevati… basementEnclosure…
    ##    <chr>                <dttm>              <chr>             <chr>             
    ##  1 FALSE                2021-07-25 00:00:00 7                 NULL              
    ##  2 FALSE                2021-07-25 00:00:00 NULL              1                 
    ##  3 FALSE                2021-07-25 00:00:00 NULL              NULL              
    ##  4 FALSE                2021-09-30 00:00:00 5                 NULL              
    ##  5 FALSE                2021-07-25 00:00:00 11                NULL              
    ##  6 FALSE                2021-07-25 00:00:00 NULL              NULL              
    ##  7 FALSE                2021-07-25 00:00:00 NULL              NULL              
    ##  8 FALSE                2021-07-25 00:00:00 NULL              NULL              
    ##  9 FALSE                2021-07-25 00:00:00 NULL              NULL              
    ## 10 FALSE                2021-07-25 00:00:00 49                NULL              
    ## # … with 36 more variables: reportedCity <chr>, condominiumIndicator <chr>,
    ## #   policyCount <chr>, countyCode <chr>, communityRatingSystemDiscount <chr>,
    ## #   dateOfLoss <dttm>, elevatedBuildingIndicator <chr>,
    ## #   elevationCertificateIndicator <chr>, elevationDifference <chr>,
    ## #   censusTract <chr>, floodZone <chr>, houseWorship <chr>, latitude <chr>,
    ## #   longitude <chr>, locationOfContents <chr>, lowestAdjacentGrade <chr>,
    ## #   lowestFloorElevation <chr>, numberOfFloorsInTheInsuredBuilding <chr>, …

There are a variety of other ways to more precisely target the data you
want to retrieve by specifying how many records you want returned,
specifying which columns in a data set to return, and applying filters
to any of the columns in a data set. For more information and examples
of use cases, see the [Getting
Started](https://github.com/dylan-turner25/rfema/blob/main/vignettes/getting_started.md)
vignette.

------------------------------------------------------------------------

Please note that `rfema` is released with a [Contributor Code of
Conduct](https://github.com/dylan-turner25/rfema/blob/main/CONTRIBUTING.md).
By contributing to the package you agree to abide by its terms.
