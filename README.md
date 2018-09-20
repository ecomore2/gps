
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
> library(magrittr) # %>% %<>%  
> library(measurements)
> library(purrr)

Attaching package: 'purrr'
The following object is masked from 'package:magrittr':

    set_names
> library(readxl)
> library(spatstat)
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
+   as.numeric(names(which(table(df[["id"]]) > 1)))
+ }
```

``` r
> are_same_points <- function(x) {
+   require(purrr)
+   invoke(identical, x$longitude) & invoke(identical, x$latitude)
+ }
```

Cleaning data
-------------

``` r
> iplserver <- read_excel("../../raw_data/GPS/IPL Server.xlsx") %>% 
+   transmute(id        = as.numeric(Reference),
+             longitude = Longitude,
+             latitude  = Latitude)
Warning in evalq(as.numeric(Reference), <environment>): NAs introduits lors
de la conversion automatique
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

Testing per file:

``` r
> lapply(list(iplserver, smartphoneapp, oldmachine, newmachine), check_duplicates)
[[1]]
[1] 7681

[[2]]
[1] 4374 5925 6060 6640

[[3]]
numeric(0)

[[4]]
numeric(0)
```

Testing for all the files put together:

``` r
> gps <- bind_rows(iplserver, smartphoneapp, oldmachine, newmachine)
> duplicates <- check_duplicates(gps)
> duplicates <- gps %>%
+   filter(id %in% duplicates) %>%
+   split(.$id) %>%
+   sapply(are_same_points) %>%
+   `!`() %>%
+   which() %>%
+   names() %>%
+   as.numeric()
> list(iplserver, smartphoneapp, oldmachine, newmachine) %>% 
+   sapply(function(x) is.element(duplicates, x$id)) %>%
+   as.data.frame() %>%
+   setNames(c("iplserver", "smartphone", "oldmachine", "newmachine")) %>%
+   `rownames<-`(duplicates)
     iplserver smartphone oldmachine newmachine
4445     FALSE       TRUE      FALSE       TRUE
4536     FALSE       TRUE      FALSE       TRUE
4582     FALSE       TRUE      FALSE       TRUE
4600     FALSE       TRUE      FALSE       TRUE
4632     FALSE       TRUE      FALSE       TRUE
4831     FALSE       TRUE      FALSE       TRUE
4840     FALSE       TRUE      FALSE       TRUE
4897     FALSE       TRUE      FALSE       TRUE
5018     FALSE       TRUE      FALSE       TRUE
5019     FALSE       TRUE      FALSE       TRUE
5033     FALSE       TRUE      FALSE       TRUE
5122     FALSE       TRUE      FALSE       TRUE
5217     FALSE       TRUE      FALSE       TRUE
5218     FALSE       TRUE      FALSE       TRUE
5251     FALSE       TRUE      FALSE       TRUE
5252     FALSE       TRUE      FALSE       TRUE
5254     FALSE       TRUE      FALSE       TRUE
5925     FALSE       TRUE      FALSE      FALSE
6060     FALSE       TRUE      FALSE      FALSE
6323     FALSE       TRUE      FALSE       TRUE
6640     FALSE       TRUE      FALSE      FALSE
7413     FALSE       TRUE      FALSE       TRUE
7436     FALSE       TRUE      FALSE       TRUE
7681      TRUE      FALSE      FALSE      FALSE
8101      TRUE      FALSE      FALSE       TRUE
```
