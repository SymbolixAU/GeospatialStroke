## —- Libraries and Census Data

``` r
library(tidyverse)
#> ── Attaching packages ────────────────────────────────── tidyverse 1.2.1 ──
#> ✔ ggplot2 3.1.0     ✔ purrr   0.2.5
#> ✔ tibble  2.0.0     ✔ dplyr   0.7.8
#> ✔ tidyr   0.8.2     ✔ stringr 1.3.1
#> ✔ readr   1.3.1     ✔ forcats 0.3.0
#> ── Conflicts ───────────────────────────────────── tidyverse_conflicts() ──
#> ✖ dplyr::filter() masks stats::filter()
#> ✖ dplyr::lag()    masks stats::lag()
library(sf)
#> Linking to GEOS 3.7.1, GDAL 2.3.2, PROJ 5.2.0
#> WARNING: different compile-time and runtime versions for GEOS found:
#> Linked against: 3.7.1-CAPI-1.11.1 27a5e771 compiled against: 3.7.0-CAPI-1.11.0
#> It is probably a good idea to reinstall sf, and maybe rgeos and rgdal too
library(units)
#> udunits system database from /usr/share/udunits
library(tmaptools)
#> Warning in fun(libname, pkgname): rgeos: versions of GEOS runtime 3.7.1-CAPI-1.11.1
#> and GEOS at installation 3.7.0-CAPI-1.11.0differ
postcodeboundariesAUS <- 
    file.path(here::here(), "ABSData", "Boundaries/POA_2016_AUST.shp") %>%
    sf::read_sf ()

basicDemographicsVIC <- file.path(here::here(), "ABSData",
                                  "2016 Census GCP Postal Areas for VIC",
                                  "2016Census_G01_VIC_POA.csv") %>%
    readr::read_csv()
#> Parsed with column specification:
#> cols(
#>   .default = col_double(),
#>   POA_CODE_2016 = col_character()
#> )
#> See spec(...) for full column specifications.
```

Clean up the demographics to only those columns that we’re interested
in. Presume just for illustrative purposes here that those are only the
basic “Age” classes. There are also columns about the ages of persons
attending educational institutions which need to be removed.

``` r
library(magrittr)
#> 
#> Attaching package: 'magrittr'
#> The following object is masked from 'package:purrr':
#> 
#>     set_names
#> The following object is masked from 'package:tidyr':
#> 
#>     extract

basicDemographicsVIC <- select(basicDemographicsVIC, POA_CODE_2016, starts_with("Age_"), -starts_with("Age_psns_"))
    
```

## —- JoinCensusAndBoundaries —-

Join the demographics and shape tables, retaining victoria only use
postcode boundaries as the reference data frame so that coordinate
reference system is retained.

``` r
basicDemographicsVIC <- right_join(postcodeboundariesAUS,
                                   basicDemographicsVIC, 
                                   by=c("POA_CODE" = "POA_CODE_2016"))
```

## —- GeocodeRehabNetwork —-

To be clean
up

``` r
rehab_addresses <- c(DandenongHospital = "Dandenong Hospital, Dandenong VIC 3175, Australia",
                     CaseyHospital = "62-70 Kangan Dr, Berwick VIC 3806, Australia",
                     KingstonHospital = "The Kingston Centre, Heatherton VIC 3202, Australia")
RehabLocations <- tmaptools::geocode_OSM(rehab_addresses, as.sf=TRUE)
```

transform rehab locations to the same reference system

``` r
RehabLocations <- sf::st_transform(RehabLocations,
                                   sf::st_crs(basicDemographicsVIC))
```

## Check geocoding

With `tmap`:

``` r
library(tmap)
tmap_mode("view")

tm_shape(RehabLocations) + tm_markers() + 
  tm_basemap("OpenStreetMap")
```

Or with `mapdeck`:

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_pointcloud (RehabLocations,
        layer_id = "rehab-locations")
```

## —- Postcodes surrounding rehab locations

There are 699 postcodes which we now want to reduce to only those within
a specified distance of the rehab locations, chosen here as 10km. Note
that we just use straight line distances here, because we only need to
roughly determine which postcodes surround our rehab centres. The
subsequent calculations will then use more accurate distances along
street networks.

``` r
dist_to_loc <- function (geometry, location){
    units::set_units(st_distance(geometry, location)[,1], km)
}
dist_range <- units::set_units(10, km)

#basicDemographicsVIC <- basicDemographicsVIC_old
basicDemographicsVIC_old <- basicDemographicsVIC
basicDemographicsVIC <- mutate(basicDemographicsVIC,
       DirectDistanceToDandenong = dist_to_loc(geometry,RehabLocations["DandenongHospital", ]),
       DirectDistanceToCasey     = dist_to_loc(geometry,RehabLocations["CaseyHospital", ]),
       DirectDistanceToKingston  = dist_to_loc(geometry,RehabLocations["KingstonHospital", ]),
       DirectDistanceToNearest   = pmin(DirectDistanceToDandenong,
                                        DirectDistanceToCasey,
                                        DirectDistanceToKingston)
)
basicDemographicsRehab <- filter(basicDemographicsVIC,
                                 DirectDistanceToNearest < dist_range) %>%
        mutate(Postcode = as.numeric(POA_CODE16)) %>%
        select(-starts_with("POA_"))
```

That reduces the data down to 47 nearby postcodes, with the last 2 lines
converting all prior postcode columns (of which there were several all
beginning with “POA”) to a single numeric column named “Postcode”.

## —- SamplePostCodes —-

Select random addresses using a geocoded database

``` r
devtools::install_github("HughParsonage/PSMA")
```

Will increase this for the real example

``` r
addressesPerPostcode <- 500
```

A special function so we can sample the postcodes as we go. Sampling
syntax is due to the use of data.table inside PSMA. The last
`st_as_sf()` command converts the points labelled “LONGITUDE” and
“LATITUDE” into `sf::POINT` objects. (This function takes a few
seconds because of the `fetch_postcodes` call.)

``` r
library(PSMA)
samplePCode <- function(pcode, number) {
  d <- fetch_postcodes(pcode)
  return(d[, .SD[sample(.N, min(number, .N))], by=.(POSTCODE)])
}

randomaddresses <- map(basicDemographicsRehab$Postcode,
                       samplePCode,
                       number=addressesPerPostcode) %>%
            bind_rows() %>%
            sf::st_as_sf(coords = c("LONGITUDE", "LATITUDE"),
                         crs=st_crs(basicDemographicsRehab),
                         agr = "constant")
```

## —- PlotSampleLocations —-

With `tmap`:

``` r
library(tmap)
tmap_mode("view")
tm_shape(randomaddresses) + tm_markers(clustering=FALSE) + 
    tm_basemap("OpenStreetMap")
head(randomaddresses)
```

or with `mapdeck`

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_pointcloud (randomaddresses,
                    radius = 2,
                    layer_id = "randomaddresses")
```

## —- AddressesToRehab —-

Compute the road distance and travel time from each address to each
hospital. This first requires a local copy of the street network within
the bounding polygon defined by `basicDemographicsRehab`. This is
easiest done with the `dodgr` package, which directly calls the
`osmdata` package to do the downloading.

### Street Network

It is instructive to examine the `mapdeck` view of the postcode
polygons:

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_polygon (basicDemographicsRehab, fill_colour="Postcode",
        layer_id = "randomaddresses")
```

The basic way to download the street network is within a defined,
implicitly rectangular, bounding box, but in this case that extends from
Mornington to St Kilda, and out to the Dandenongs, and even Koo Wee
Rup\! It is much better to extract the street network only within the
polygon defining our nearby postcode areas, which first needs to be
re-projected onto the CRS of OpenStreetMap data, which is epsg4326.
`st_union` merges all of the polygons to form the single enclosing
polygon, and the final command simply extracts the longitudinal and
latitudinal coordinates of that polygon (rather than leaving them in
`sf` format).

``` r
bounding_polygon <- sf::st_transform(basicDemographicsRehab,
                                     sf::st_crs(4326)) %>%
    sf::st_union () %>%
    sf::st_coordinates ()
bounding_polygon <- bounding_polygon [, 1:2]
```

We can now download the street network enclosed within that polygon.
Note that this is still a rather large network - over 40MB of data
representing over 60,000 street sections - that might take a minute or
two to process. It is therefore easier to save the result to disc for
quicker re-usage.

``` r
library(dodgr)
system.time (
dandenong_streets <- dodgr_streetnet (bounding_polygon, expand = 0, quiet = FALSE)
)
saveRDS (dandenong_streets, file = "dandenong-streets.Rds")
format (file.size ("dandenong-streets.Rds"), big.mark = ",")
nrow (dandenong_streets)
```

The network can then be re-loaded with

``` r
dandenong_streets <- readRDS ("dandenong-streets.Rds")
```

### Distances to Hospitals

The `dodgr` package needs to de-compose the `sf`-formatted street
network, which consists of long, connected road segments, into
individual edges. This is done with the `weight_streetnet()` function,
which modifies the distance of each edge to reflect typical travel
conditions for a nominated mode of transport.

``` r
library (dodgr)
dandenong_streets <- readRDS ("dandenong-streets.Rds")
net <- weight_streetnet (dandenong_streets, wt_profile = "motorcar")
#> The following highway types are present in data yet lack corresponding weight_profile values: raceway, construction, corridor, road, proposed, living_street, bus_stop, NA,
nrow (dandenong_streets); nrow (net)
#> [1] 62624
#> [1] 626391
```

The resultant network has a `d_weighted` column which preferentially
weights the distances for the nominated mode of tranport. Those parts of
the network which are unsuitable for vehicular transport have values of
`.Machine$double.xmax =` 1.797693110^{308}. Because we want to align our
random points to the *routable* component of the network, these need to
be removed.

``` r
net <- net [which (net$d_weighted < .Machine$double.xmax), ]
nrow (net)
#> [1] 437145
```

The 62624 streets have been converted to 437145 distinct edges. We can
now use the `net` object to calculate the distances, along with simple
numeric coordinates of our routing points, projected on to the same CRS
as OpenStreetMap (OSM), which is
4326:

``` r
fromCoords <- st_coordinates (st_transform (randomaddresses, crs = 4326))
toCoords <- st_coordinates (st_transform (RehabLocations, crs = 4326))
```

Although not necessary, distance calculation is quicker if we map these
`from` and `to` points precisely on to the network itself. OSM assigns
unique identifiers to every single object, and so our routing
coordinates can be converted to OSM identifiers of the nearest street
nodes. The nodes themselves are obtained with the `dodgr_vertices()`
function.

``` r
nodes <- dodgr_vertices (net)
fromIDX <- match_pts_to_graph (nodes, fromCoords, connected = TRUE)
from <- unique (nodes$id [fromIDX])
to <- nodes$id [match_pts_to_graph (nodes, toCoords, connected = TRUE)]
```

The matrices of `from` and `to` coordinates have now been converted to
simple vectors of OSM identifiers.

``` r
system.time (
             d <- dodgr_dists (net, from = from, to = to)
)
#>    user  system elapsed 
#>   0.890   0.000   0.803
```

And that takes only around 1 second to calulate distances between (3
rehab centres times 20,000 random addresses = ) 60,000 pairs of points.

## —- CatchmentBasins —-

First assign each point to its nearest hospital according to the street
network distances returned from `dodgr_dists`. Note that points on the
outer periphery of the network may not necessarily be connected, as
we’ll see below.

``` r
DestNames <- c(rownames(RehabLocations), "Disconnected")
# assign each source address to the nearest destination
DestNumber <- as.numeric (apply(d, MARGIN=1, which.min))
DestNumber [is.na (DestNumber)] <- 4 # the disconnected points
BestDestination <- DestNames[DestNumber]
table (BestDestination)
#> BestDestination
#>     CaseyHospital DandenongHospital      Disconnected  KingstonHospital 
#>              4230              6467                99              8889
```

And there are 99 points that are not connected. The allocation of
points, including these disconnected ones, can be inspected on a map
with the following code, start by setting up a `data.frame` of
`fromCoords`.

``` r
fromCoords <- nodes [match (from, nodes$id), ]
fromCoords$DestNumber <- DestNumber
fromCoords$Destination <- BestDestination
```

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 10) %>%
    add_pointcloud (data = fromCoords, lon = "x", lat = "y",
                    fill_colour = "DestNumber",
                    palette = "plasma")
```

As a final step, we’ll convert those clusters of points into enclosing
polygons, using a Voronoi tesselation. `sf::st_voronoi` doesn’t return
the polygons in the same order as the original points, requiring a
manual re-sorting in order to use this to match voronoi polygons to
points for each catchment.

``` r
g <- st_multipoint(as.matrix(fromCoords[,c("x", "y")]))
v <- st_voronoi(x=g) # results in geometry collection objects
v <- st_collection_extract(v) # converts to polygons
fromCoords_sf <- st_as_sf(fromCoords, coords=c("x", "y"))
vorder <- unlist(st_intersects(fromCoords_sf, v))
v <- v[vorder] # polygons in same order as points
v <- st_sf (DestNumber = fromCoords$DestNumber,
            Destination = fromCoords$Destination,
            geometry = v,
            crs = 4326)
```

We then just need to filter those Voronoi polygons associated with each
catchment, and extract the surrounding polygons. (This can take quite
some time.)

``` r
bounding_polygon <- sf::st_transform(basicDemographicsRehab,
                                     sf::st_crs(4326)) %>%
  sf::st_union () 
v <- lapply (1:3, function (i) {
                 v [v$DestNumber == i, ] %>%
                     st_intersection (bounding_polygon) %>%
                     st_union() })
v <- st_sf (DestNumber = 1:3,
            Destination = DestNames [1:3],
            geometry = do.call (c, v))
```

Then plot with `mapdeck`:

``` r
mapdeck(location = c(145.2, -38),
        zoom = 10) %>%
    add_polygon (data = v,
                    fill_colour = "DestNumber",
                    fill_opacity = 150,
                    palette = "plasma") %>%
    add_pointcloud (data = RehabLocations)
```

## —- CasesPerCentre —-

Also need a per postcode breakdown of proportion of addresses going to
each centre, so that we can compute the number of cases going to each
centre. The above procedure occasionally mapped multiple addresses onto
the same network points. As we were only interested in the enclosing
polygons, these repeated points were removed. In the present case,
however, these repeats need to be counted, so we need to go back to
where we were before, which is the `randomaddresses`.

``` r
dim (randomaddresses); dim (fromCoords)
#> [1] 28000    14
#> [1] 19685     7
length (from); length (DestNumber)
#> [1] 19685
#> [1] 19685
```

We need to repeat the calculation of `DestNumber` using the full set of
`fromCoords`, including those repeatedly matched onto same network
points.

``` r
fromCoords <- st_coordinates (st_transform (randomaddresses, crs = 4326))
fromIDX <- match_pts_to_graph (nodes, fromCoords, connected = TRUE)
from <- nodes$id [fromIDX]
to <- nodes$id [match_pts_to_graph (nodes, toCoords, connected = TRUE)]
d <- dodgr_dists (net, from = from, to = to)
DestNames <- c(rownames(RehabLocations), "Disconnected")
DestNumber <- as.numeric (apply(d, MARGIN=1, which.min))
DestNumber [is.na (DestNumber)] <- 4 # the disconnected points
BestDestination <- DestNames[DestNumber]
```

The following lines then group the above data by both postcode and
destination hospital.

``` r
postcodes <- data.frame (POSTCODE = randomaddresses$POSTCODE,
                         DestNumber = DestNumber,
                         Destination = BestDestination,
                         stringsAsFactors = FALSE) %>%
    group_by (POSTCODE, DestNumber, Destination) %>%
    summarise (n = length (DestNumber))
postcodes
#> # A tibble: 87 x 4
#> # Groups:   POSTCODE, DestNumber [?]
#>    POSTCODE DestNumber Destination           n
#>       <int>      <dbl> <chr>             <int>
#>  1     3144          3 KingstonHospital    478
#>  2     3144          4 Disconnected         22
#>  3     3145          3 KingstonHospital    500
#>  4     3146          3 KingstonHospital    453
#>  5     3146          4 Disconnected         47
#>  6     3147          3 KingstonHospital    498
#>  7     3147          4 Disconnected          2
#>  8     3148          3 KingstonHospital    500
#>  9     3149          1 DandenongHospital    36
#> 10     3149          3 KingstonHospital    461
#> # … with 77 more rows
```