
<!--
IMAGES:
Insert them with: ![alt text](image.png)
You can also resize them if needed: convert image.png -resize 50% image.png
If you want to center the image, go through HTML code:
<div style="text-align:center"><img src ="image.png"/></div>

REFERENCES:
For references: Put all the bibTeX references in the file "references.bib"
in the current folder and cite the references as @key or [@key] in the text.
Uncomment the bibliography field in the above header and put a "References"
title wherever you want to display the reference list.
-->
Preambule
---------

There are 4 files containing GPS coordinates data in the `raw_data/GPS` folder:

-   `Old Machine 2012-2015.xls` that contains data collected by the IPL staff using the old GPS device;
-   `New Machine2015-2018.xls` that contains data collected by the IPL staff using the new GPS device;
-   `Shared with app.xls` that contains data sent by patients through WhatsApp;
-   `IPL Server.xlsx` that contains data sent by patients or volunteers from the municipality, using the web application developped by IPL.

Cleaned data are merged with PACS data and written both in CSV and RDS in `cleaned_data/pacs.csv` and `cleaned_data/pacs.rds`.

Packages
--------

Packages currently installed on the system:

``` r
> installed_packages <- rownames(installed.packages())
```

Packages that we need from [CRAN](https://cran.r-project.org):

``` r
> cran <- c("devtools",     # development tools
+           "dplyr",        # data frames manipulation
+           "geosphere",    # spherical trigonometry
+           "magrittr",     # pipe operators
+           "measurements", # tools for units measurement
+           "purrr",        # functional programming tools
+           "sp",           # classes and methods for spatial data
+           "readxl",       # to read excel files
+           "spatstat"      # spatial point pattern analysis
+           )
```

Installing these packages when not already installed:

``` r
> to_install <- !cran %in% installed_packages
> if (any(to_install)) install.packages(cran[to_install])
```

We additionally need the `ecomore` package from [GitHub](https://github.com/ecomore2/ecomore):

``` r
> if (! "ecomore" %in% installed_packages)  devtools::install_github("ecomore2/ecomore")
```

Loading the packages for interactive use at the command line:

``` r
> invisible(lapply(c(setdiff(cran, "devtools"), "ecomore"), library, character.only = TRUE))
```

Utilitary functions
-------------------

The following function convert DMS coordinates into decimal degrees coordiantes:

``` r
> dms2dd <- function(x) {
+   require(magrittr)     # %>% 
+   require(measurements) # conv_unit
+   x %>%
+     sub("°", " ", .) %>%
+     sub("'", " ", .) %>%
+     sub("\\.*''[EN]", "", .) %>%
+     conv_unit("deg_min_sec", "dec_deg")
+ }
```

The following function identifies in a vector of coordinates the ones that are in DMS and then uses the above function to convert them into decimal degrees coordinates:

``` r
> dms2dd2 = function(x) {
+  sel <- which(grepl("[EN]", x))
+  x[sel] <- sapply(x[sel], dms2dd)
+  as.numeric(x)
+ }
```

The following function reads and reformat GPS coordinates data collected from one of the 2 GPS devices (the old and the new one) as well as from the WhatsApp app:

``` r
> read_gps <- function(file, tab, tabs) {
+   require(dplyr)  # %>%, transmute
+   require(readxl) # read_excel
+   read_excel(file, grep(tab, tabs, TRUE, value = TRUE)) %>%
+     transmute(id        = as.integer(`N° patient`),
+               longitude = dms2dd2(sub("/",".", Longitude)),
+               latitude  = dms2dd2(Latitude))
+ }
```

The following function returns the ID that have more than GPS coordinate record:

``` r
> check_duplicates <- function(df) {
+   as.numeric(names(which(table(df[["id"]]) > 1)))
+ }
```

The following function takes a data frame with 2 columns, longitude and latitude and two rows (2 points) and tests whether these 2 points have exactly the same coordinates:

``` r
> are_same_points <- function(x) {
+   require(purrr) # invoke
+   invoke(identical, x$longitude) & invoke(identical, x$latitude)
+ }
```

The following function takes a data frame of coordinates and ID, and the ID that has more than one entry in the data frame, and returns the distance in km between these two entries:

``` r
> dist_btw_2_houses <- function(df, i) {
+   require(dplyr)     # %>%, filter, select
+   require(geosphere) # distHarversine
+   tmp <- df %>%
+     filter(id == i) %>%
+     select(-id, -source) %>%
+     as.matrix()
+   distHaversine(tmp[1, ], tmp[2, ]) / 1000
+ }
```

Reading data
------------

Reading the GPS coordinates from the 4 sources:

``` r
> file <- "../../raw_data/GPS/all main data.xls"
> tabs <- excel_sheets(file)
> oldmachine <- read_gps(file, "old", tabs)
> newmachine <- read_gps(file, "new", tabs)
> smartphoneapp <- read_gps(file, "app", tabs)
> iplserver <- read_excel(file, grep("server", tabs, TRUE, value = TRUE)) %>% 
+              transmute(id        = as.integer(Reference),
+                        longitude = Longitude,
+                        latitude  = Latitude)
Warning in evalq(as.integer(Reference), <environment>): NAs introduits lors
de la conversion automatique
```

Testing for duplicates in each of the 4 files, and fixing
---------------------------------------------------------

``` r
> lapply(list(iplserver, smartphoneapp, oldmachine, newmachine), check_duplicates) %>% 
+   setNames(c("iplserver", "smartphoneapp", "oldmachine", "newmachine"))
$iplserver
[1] 7681

$smartphoneapp
[1] 4374 5925 6060 6640

$oldmachine
numeric(0)

$newmachine
numeric(0)
```

Note that patients `7681` and `6060` live in 2 different houses at the same time (see issue [\#1](https://github.com/ecomore2/gps/issues/1)), hence the reason for their duplication. Patients `4374` and `6640` sent the GPS data twice (see issue [\#1](https://github.com/ecomore2/gps/issues/1)), so we can delete the first coordinates for the 2 of them. Patient `5925` first reported the house only and the coordinates were retrieved from Google Maps; then he sent the GPS coordinates (see issue [\#1](https://github.com/ecomore2/gps/issues/1)). So we can delete the first coordinates for this one too:

``` r
> to_delete <- sapply(c(4374, 5925, 6640),
+                     function(x) which(smartphoneapp$id == x)[1])
> smartphoneapp <- smartphoneapp[-to_delete, ]
```

Merging the 4 sources of data
-----------------------------

``` r
> gps <- bind_rows(server   = iplserver,
+                  whatsapp = smartphoneapp,
+                  old_gps  = oldmachine,
+                  new_gps  = newmachine, .id = "source")
```

Looking for GPS coordinates that have been taken by more than one mean:

``` r
> duplicates <- check_duplicates(gps)
```

Here we remove the duplicates that are exactly the same points:

``` r
> duplicates <- gps %>%
+   filter(id %in% duplicates) %>%
+   split(.$id) %>%
+   sapply(are_same_points) %>%
+   `!`() %>%
+   which() %>%
+   names() %>%
+   as.numeric()
```

Now we look to what sources these duplicates are:

``` r
> duplicates <- list(iplserver, smartphoneapp, oldmachine, newmachine) %>% 
+   sapply(function(x) is.element(duplicates, x$id)) %>%
+   as.data.frame() %>%
+   setNames(c("server", "whatsapp", "oldgps", "newgps")) %>%
+   mutate(id = duplicates) %>% 
+   select(id, everything())
> duplicates
     id server whatsapp oldgps newgps
1  4445  FALSE     TRUE  FALSE   TRUE
2  4536  FALSE     TRUE  FALSE   TRUE
3  4582  FALSE     TRUE  FALSE   TRUE
4  4600  FALSE     TRUE  FALSE   TRUE
5  4632  FALSE     TRUE  FALSE   TRUE
6  4831  FALSE     TRUE  FALSE   TRUE
7  4840  FALSE     TRUE  FALSE   TRUE
8  4897  FALSE     TRUE  FALSE   TRUE
9  5018  FALSE     TRUE  FALSE   TRUE
10 5019  FALSE     TRUE  FALSE   TRUE
11 5033  FALSE     TRUE  FALSE   TRUE
12 5122  FALSE     TRUE  FALSE   TRUE
13 5217  FALSE     TRUE  FALSE   TRUE
14 5218  FALSE     TRUE  FALSE   TRUE
15 5251  FALSE     TRUE  FALSE   TRUE
16 5252  FALSE     TRUE  FALSE   TRUE
17 5254  FALSE     TRUE  FALSE   TRUE
18 6060  FALSE     TRUE  FALSE  FALSE
19 6323  FALSE     TRUE  FALSE   TRUE
20 7413  FALSE     TRUE  FALSE   TRUE
21 7436  FALSE     TRUE  FALSE   TRUE
22 7681   TRUE    FALSE  FALSE  FALSE
23 8101   TRUE    FALSE  FALSE   TRUE
```

So, we have 20 houses that were checked both with WhatsApp and the server:

``` r
> duplicates %>%
+   filter(whatsapp, newgps) %>% 
+   nrow()
[1] 20
```

One house that was checked both with the server and by the GPS:

``` r
> duplicates %>%
+   filter(server, newgps) %>% 
+   nrow()
[1] 1
```

And the 2 patients that live in 2 houses (see above):

``` r
> names(which(rowSums(duplicates) < 2))
NULL
```

Patients 4445, 4536, 4582, 4600, 4632, 4831, 4840, 4897, 5018, 5019, 5033, 5122, 5217, 5218, 5251, 5252, 5254, 6323, 7413 and 7436 had their coordinates reported both on-site from GPS device and through the smartphone app in order to assess the accuracy of the latter (see issue \#3). Thus we can remove these ID from the smartphone data. But, before that, we can look at the distances between the two means of data collection. Let's first get the ID of these duplicates:

``` r
> dpl <- duplicates %>%
+   filter(whatsapp, newgps) %>%
+   select(id) %>%
+   unlist() %>%
+   unname()
```

The following function makes a matrix of coordinates as required by the function `distHaversine`:

``` r
> make_coord_mat <- function(df, idsel, sourcesel) {
+   df %>%
+     filter(id %in% idsel, source == sourcesel) %>% 
+     arrange(id) %>%  # to make sure that the coordinates are in the same order
+     select(longitude, latitude) %>%  # in the two matrices
+     as.matrix()
+ }
```

``` r
> dpl_nw <- make_coord_mat(gps, dpl, "new_gps")
> dpl_sp <- make_coord_mat(gps, dpl, "whatsapp")
> sort(distHaversine(dpl_nw, dpl_sp)) / 1000
 [1]  0.009745362  0.010946436  0.011268729  0.030684530  0.031444077
 [6]  0.032671816  0.043119851  0.050385576  0.102365185  0.116229296
[11]  0.133134439  0.158631137  0.264623458  0.288712412  0.317148231
[16]  0.704071464  1.345591159  6.081439467 49.873441728 50.627912191
```

Let's remove all the whatsapp data that are also collected by GPS, as well as the server datum that is also collected by GPS:

``` r
> whatsapp_dplcts <- duplicates %>%
+   filter(whatsapp, newgps) %>%
+   select(id) %>%
+   unlist()
> gps %<>% filter(!(id %in% whatsapp_dplcts & source == "whatsapp"))
```

Let's do the same with the server datum. For ID 8101, the coordinates were collected on-site with GPS device by mistake, forgetting that the patient had aleady sent the data through the server (see issue [\#1](https://github.com/ecomore2/gps/issues/1)). Let's first look at the distance:

``` r
> ip <- make_coord_mat(gps, "8101", "server")
> nm <- make_coord_mat(gps, "8101", "new_gps")
> distHaversine(ip, nm) / 1000
[1] 0.03922978
```

And then we remove the server datum:

``` r
> server_dplcts <- duplicates %>%
+   filter(server, newgps) %>%
+   select(id) %>%
+   unlist()
> gps %<>% filter(!(id %in% server_dplcts & source == "server"))
```

We can check that the only 2 duplicated IDs and the ones that live in two different houses:

``` r
> gps %>%
+   select(id) %>%
+   table() %>%
+   `>`(1) %>%
+   which() %>%
+   names()
[1] "6060" "7681"
```

We can fancy looking at the distances, in km, between these two houses:

``` r
> dist_btw_2_houses(gps, 6060)
[1] 7.052577
> dist_btw_2_houses(gps, 7681)
[1] 9.753693
```

Let's check whether there are missing values in the coordinates:

``` r
> filter(gps, is.na(longitude) | is.na(latitude))
# A tibble: 2 x 4
  source      id longitude latitude
  <chr>    <int>     <dbl>    <dbl>
1 whatsapp  6983        NA       NA
2 new_gps     NA        NA       NA
```

Let's remove the points with missing values in the coordinates:

``` r
> gps %<>% na.exclude()
```

Checking that GPS coordinates are in Vientiane Capital
------------------------------------------------------

Let's get the polygons of the provinces of Lao PDR:

``` r
> provinces <- readRDS("../../cleaned_data/gadm/gadm36_LAO_1_sp.rds")
```

Let's extract the province of Vientiane Capital:

``` r
> vientiane <- subset(provinces, grepl("Vientiane", provinces$VARNAME_1))
```

Let's identify the GPS points that are not inside the polygon of the province of Vientiane Capital:

``` r
> not_in_vc <- gps %>%
+   select(-id, -source) %>% 
+   SpatialPoints(CRS(proj4string(vientiane))) %>% 
+   over(vientiane) %>% 
+   `[`(, 1) %>% 
+   is.na() %>% 
+   which()
> gps[not_in_vc, ]
# A tibble: 10 x 4
   source      id longitude latitude
   <chr>    <int>     <dbl>    <dbl>
 1 whatsapp  5553   103.        18.3
 2 whatsapp  6019   102.        18.0
 3 whatsapp  7004     0.926     40.4
 4 whatsapp  7136   103.        21.5
 5 new_gps   3648   103.       103. 
 6 new_gps   3459   103.       103. 
 7 new_gps   3460   103.       103. 
 8 new_gps   3698   103.       103. 
 9 new_gps   4100   103.        18.2
10 new_gps   4354   103.        18.2
```

Those for the which the latitude is above 100 actually have the longitude instead of the latitude:

``` r
> gps[not_in_vc, ] %>%
+   filter(latitude > 100) %$%
+   identical(longitude, latitude)
[1] TRUE
```

Removing them leaves:

``` r
> filter(gps[not_in_vc, ], latitude < 100)
# A tibble: 6 x 4
  source      id longitude latitude
  <chr>    <int>     <dbl>    <dbl>
1 whatsapp  5553   103.        18.3
2 whatsapp  6019   102.        18.0
3 whatsapp  7004     0.926     40.4
4 whatsapp  7136   103.        21.5
5 new_gps   4100   103.        18.2
6 new_gps   4354   103.        18.2
```

ID `7004` is obviously a mistake, let's remove it too:

``` r
> out_of_vc <- filter(gps[not_in_vc, ], latitude < 100, longitude > 100)
```

And let's see where these points left are on the map:

``` r
> plot(provinces, col = "grey")
> with(out_of_vc, points(longitude, latitude, col = "red"))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-38-1.png" width="407.736" style="display: block; margin: auto;" />

`7136` is from **Phôngsali province**! The other ones are actually very close to Vientiane capital:

``` r
> plot(vientiane, col = "grey")
> with(out_of_vc, points(longitude, latitude, col = "red"))
```

<img src="README_files/figure-markdown_github/unnamed-chunk-39-1.png" width="407.736" style="display: block; margin: auto;" />

Merging with PACS and writting to disk
--------------------------------------

``` r
> file <- "../../cleaned_data/pacs.rds"
> if (file.exists(file)) {
+   pacs_file <- readRDS(file)
+   if ("source" %in% names(pacs_file)) {
+     full_join(select(pacs_file, -source, -longitude, -latitude), gps, "id") %>% 
+       write2disk("cleaned_data", "pacs")
+   } else {
+     full_join(pacs_file, gps, "id") %>% 
+       write2disk("cleaned_data", "pacs")
+   }
+ } else write2disk(pacs, "cleaned_data", "gps")
```
