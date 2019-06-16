
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

# GPS data

There are 4 files containing GPS coordinates data in the `raw_data/GPS`
folder:

  - `Old Machine 2012-2015.xls` that contains data collected by the IPL
    staff using the old GPS device;
  - `New Machine2015-2018.xls` that contains data collected by the IPL
    staff using the new GPS device;
  - `Shared with app.xls` that contains data sent by patients through
    WhatsApp;
  - `IPL Server.xlsx` that contains data sent by patients or volunteers
    from the municipality, using the web application developped by IPL.

The data cleaned by this
[pipeline](https://ecomore2.github.io/gps/make_data.html), are saved to
the
[`data\pacs.csv`](https://raw.githubusercontent.com/ecomore2/gps/master/data/gps.csv)
CSV file that can be copied and paste to a text file on your computer or
downloaded directly from R into a data
frame:

``` r
> if (! "readr" %in% rownames(installed.packages())) install.packages("readr")
> pacs <- readr::read_csv("https://raw.githubusercontent.com/ecomore2/gps/master/data/gps.csv", col_types = "icdd")
```

This [summary](https://ecomore2.github.io/gps/summarize_data.html)
provides a real-time overview of the current state of the PACS data set,
highlighting problems that remain to be fixed.
