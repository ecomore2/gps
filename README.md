
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

Packages
--------

``` r
> library(dplyr)

Attaching package: 'dplyr'
The following objects are masked from 'package:stats':

    filter, lag
The following objects are masked from 'package:base':

    intersect, setdiff, setequal, union
> library(measurements)
> library(readxl)
> library(spatstat)
Loading required package: spatstat.data
Loading required package: nlme

Attaching package: 'nlme'
The following object is masked from 'package:dplyr':

    collapse
Loading required package: rpart

spatstat 1.55-1       (nickname: 'Gamble Responsibly') 
For an introduction to spatstat, type 'beginner' 

Note: spatstat version 1.55-1 is out of date by more than 5 months; we recommend upgrading to the latest version.
```

Utilitary functions
-------------------

``` r
> dms2dd <- function(x) {
+   x %>%
+     sub("°", " ", .) %>%
+     sub("'", " ", .) %>%
+     sub("''[EN]", "", .) %>%
+     conv_unit("deg_min_sec", "dec_deg")
+ }
```

``` r
> dms2dd2 = function(x) {
+  sel <- which(grepl("[EN]", x))
+  x[sel] <- sapply(x[sel], dms2dd)
+  x
+ }
```

``` r
> read_from_machine <- function(x) {
+   x %>%
+     read_excel() %>%
+     transmute(id        = `N° patient`,
+               longitude = Longitude,
+               latitude  = Latitude)
+ }
```

``` r
> check_duplicates <- function(df) {
+   names(which(table(df[["id"]]) > 1))
+ }
```

Cleaning data
-------------

``` r
> iplserver <- read_excel("../../raw_data/GPS/IPL Server.xlsx") %>% 
+   transmute(id        = Reference,
+             longitude = Longitude,
+             latitude  = Latitude)
```

``` r
> smartphoneapp <- read_excel("../../raw_data/GPS/Shared with app.xls") %>% 
+   transmute(id        = `N° patient`,
+             longitude = as.numeric(dms2dd2(sub("/",".", Longitude))),
+             latitude  = as.numeric(dms2dd2(Latitude)))
Warning in split(as.numeric(unlist(strsplit(x_na_free, " "))) * c(3600, :
NAs introduits lors de la conversion automatique
```

``` r
> oldmachine <- read_from_machine("../../raw_data/GPS/Old Machine 2012-2015.xls")
> newmachine <- read_from_machine("../../raw_data/GPS/New Machine2015-2018.xls")
```

Testing:

``` r
> check_duplicates(iplserver)
[1] "7681"
> check_duplicates(smartphoneapp)
[1] "4374" "5925" "6060" "6640"
> check_duplicates(oldmachine)
character(0)
> check_duplicates(newmachine)
character(0)
```

Temporary fix
