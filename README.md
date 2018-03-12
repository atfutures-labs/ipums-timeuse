<!-- README.md is generated from README.Rmd. Please edit that file -->
This repository contains just one script used to analyse changes in transport behaviour of time. The data were obtained from the [Multinational Time Use Study (MTUS)](https://www.ipums.org/timeuse.shtml) data, part of the [Integrated Public Use Microdata Series (IPUMS)](https://www.ipums.org). MTUS data are jointly hosted by the Minnesota Population Center at the University of Minnesota, and the [Centre for Time Use Research](https://www.timeuse.org/mtus) at the University of Oxford.

Data downloads are accompanied by a corresponding "ddi" file, included here both in [`.xml`](./data/mtus_00001.xml) and [plain text](./data/mtus_00001.cbk) formats. The data download can be reproduced by following the structure of these files.

The downloaded data include the following variables and corresponding codes (where "MAIN" describes the main activity):

| variable | code | description                      |
|----------|------|----------------------------------|
| MAIN     | 043  | walking                          |
| MAIN     | 044  | cycling                          |
| MAIN     | 063  | travel to/from work              |
| MAIN     | 064  | education travel                 |
| MAIN     | 065  | voluntary/civic/religious travel |
| MAIN     | 066  | child/adult care travel          |
| MAIN     | 067  | shop, person/child care travel   |
| MAIN     | 068  | other travel                     |
| MTRAV    | 01   | car                              |
| MTRAV    | 02   | public transport                 |
| MTRAV    | 03   | walk/on foot                     |
| MTRAV    | 04   | other physical transport         |
| MTRAV    | 05   | other/unspecified                |

Reading the data
----------------

IPUMS data can now be read directly into **R** using the fabulous [`ipumsr` package](https://github.com/mnpopcenter/ipumsr). The primary functions simply need the "ddi" file describing the data structure. It is presumed here that the data themselves reside in the same directory. (They are not included in this repository because of their very large size).

``` r
library (ipumsr)
dat <- read_ipums_micro_list (ddi = "./data/mtus_00001.xml",
                              vars = c ("COUNTRY", "YEAR", "AGE", "SEX",
                                        "TIME", "MAIN", "MTRAV"))
#> Use of data from MTUS-X is subject to conditions including that users should
#> cite the data appropriately. Use command `ipums_conditions()` for more details.
#> 
#> Reading data...
#> Parsing data...
dat
#> $PERSON
#> # A tibble: 419,686 x 5
#>    RECTYPE   COUNTRY    YEAR   AGE SEX      
#>    <dbl+lbl> <chr+lbl> <int> <dbl> <int+lbl>
#>  1 2         AT         1992   41. 1        
#>  2 2         AT         1992   40. 2        
#>  3 2         AT         1992   70. 1        
#>  4 2         AT         1992   68. 2        
#>  5 2         AT         1992   78. 1        
#>  6 2         AT         1992   71. 2        
#>  7 2         AT         1992   74. 1        
#>  8 2         AT         1992   73. 2        
#>  9 2         AT         1992   58. 2        
#> 10 2         AT         1992   29. 2        
#> # ... with 419,676 more rows
#> 
#> $ACTIVITY
#> # A tibble: 9,224,050 x 6
#>    RECTYPE   COUNTRY    YEAR  TIME MAIN      MTRAV    
#>    <dbl+lbl> <chr+lbl> <int> <dbl> <int+lbl> <int+lbl>
#>  1 3         AT         1992  390. 2         -7       
#>  2 3         AT         1992   15. 4         -7       
#>  3 3         AT         1992   15. 66        1        
#>  4 3         AT         1992   15. 6         -7       
#>  5 3         AT         1992   15. 6         -7       
#>  6 3         AT         1992   45. 6         -7       
#>  7 3         AT         1992   30. 19        -7       
#>  8 3         AT         1992   15. 25        5        
#>  9 3         AT         1992   30. 24        -7       
#> 10 3         AT         1992   15. 21        5        
#> # ... with 9,224,040 more rows
```

Note that the `dat$ACTIVITY$TIME` variable quantifies the duration of a travel activity, and is the key variable we will be analysing here. These data include over 9 million activities from nearly 500,000 individuals from these countries:

``` r
countries <- unique (dat$PERSON$COUNTRY)
countries
#> [1] "AT" "CA" "FI" "FR" "NL" "ES" "UK" "US"
```

The two tables represent the hierarchical structure of the data. These data can then be filtered down to only those records in which people walked, rode bicycles, or used cars. We will only use the `ACTIVITY` table here.

``` r
act <- dplyr::filter (dat$ACTIVITY, MTRAV > 0 | MAIN %in% c (43:44, 63:68))
act
#> # A tibble: 1,475,679 x 6
#>    RECTYPE   COUNTRY    YEAR  TIME MAIN      MTRAV    
#>    <dbl+lbl> <chr+lbl> <int> <dbl> <int+lbl> <int+lbl>
#>  1 3         AT         1992   15. 66        1        
#>  2 3         AT         1992   15. 25        5        
#>  3 3         AT         1992   15. 21        5        
#>  4 3         AT         1992   15. 23        5        
#>  5 3         AT         1992  135. 59        5        
#>  6 3         AT         1992   15. 4         5        
#>  7 3         AT         1992   30. 68        1        
#>  8 3         AT         1992   15. 63        1        
#>  9 3         AT         1992   15. 63        3        
#> 10 3         AT         1992   15. 63        5        
#> # ... with 1,475,669 more rows
```

We still have almost 1.5 million travel activities. Now let's make equivalent tables for each activity, plus one for all travel activities combined (represented by `MTRAV > 0`):

``` r
bike <- dplyr::filter (act, MAIN == 44)
walk <- dplyr::filter (act, MAIN == 43)
allmodes <- dplyr::filter (act, MTRAV > 0)
```

Then aggregate those data to generate total durations of each activity for each year and country. The following function converts those durations to proportional times, enabling proportions of cycling and walking to be directly compared in relation to total travel time. (The latter is admittedly a rough estimate

``` r
require (magrittr)
#> Loading required package: magrittr
require (tibble)
#> Loading required package: tibble
gettimes <- function (ci = "CA")
{
    tb <- bike %>%
        dplyr::filter (COUNTRY == ci) %>%
        dplyr::group_by (YEAR) %>%
        dplyr::summarize (time = sum (TIME))
    tw <- walk %>%
        dplyr::filter (COUNTRY == ci) %>%
        dplyr::group_by (YEAR) %>%
        dplyr::summarize (time = sum (TIME))
    ta <- allmodes %>%
        dplyr::filter (COUNTRY == ci) %>%
        dplyr::group_by (YEAR) %>%
        dplyr::summarize (time = sum (TIME))
    if (nrow (tb) == 0)
        message (ci, ": no bike data")
    if (nrow (tw) == 0)
        message (ci, ": no walking data")
    year <- sort (unique (c (tb$YEAR, tw$YEAR, ta$YEAR)))
    if (length (year) < 2)
        message (ci, ": no multi-year data")
    year <- year [which (year %in% ta$YEAR)]

    times <- tibble (year = year,
                     walk = tw$time [match (year, tw$YEAR)],
                     bike = tb$time [match (year, tb$YEAR)],
                     allmodes = ta$time [match (year, ta$YEAR)])
    times [is.na (times)] <- 0
    times$walk <- times$walk / times$allmodes
    times$bike <- times$bike / times$allmodes
    return (times)
}
times <- lapply (countries, gettimes)
#> AT: no multi-year data
#> CA: no multi-year data
#> FI: no multi-year data
#> FR: no bike data
#> NL: no bike data
#> NL: no walking data
#> ES: no bike data
names (times) <- countries
```

Most countries do not unfortunately have sufficient data to analyse further, leaving only the UK and the US.

``` r
times <- times [which (countries %in% c ("UK", "US"))]
times
#> $UK
#> # A tibble: 8 x 4
#>    year   walk    bike allmodes
#>   <int>  <dbl>   <dbl>    <dbl>
#> 1  1974 0.136  0.       595280.
#> 2  1975 0.0755 0.       748440.
#> 3  1983 0.108  0.00254  395955.
#> 4  1984 0.0934 0.00384  304575.
#> 5  1987 0.0708 0.00312 1032150.
#> 6  2000 0.0964 0.0135  1065800.
#> 7  2001 0.104  0.0153  1269290.
#> 8  2005 0.134  0.00967  418660.
#> 
#> $US
#> # A tibble: 18 x 4
#>     year    walk    bike allmodes
#>    <int>   <dbl>   <dbl>    <dbl>
#>  1  1985 0.0305  0.00888  317374.
#>  2  1992 0.      0.        67275.
#>  3  1993 0.      0.       425323.
#>  4  1994 0.      0.       364896.
#>  5  1995 0.      0.        66477.
#>  6  1998 0.      0.       118011.
#>  7  1999 0.00438 0.        65102.
#>  8  2000 0.00533 0.        75960.
#>  9  2003 0.0312  0.00473 1893502.
#> 10  2004 0.0285  0.00464 1186800.
#> 11  2005 0.0300  0.00535 1132460.
#> 12  2006 0.0307  0.00608 1116563.
#> 13  2007 0.0348  0.00599 1039728.
#> 14  2008 0.0300  0.00738 1065144.
#> 15  2009 0.0334  0.00621 1119126.
#> 16  2010 0.0315  0.00646 1178886.
#> 17  2011 0.0355  0.00699 1082362.
#> 18  2012 0.0401  0.00751 1060431.
```

The US also only has usable data after 1998, so

``` r
times$US <- dplyr::filter (times$US, year > 1998)
```

Then convert these to a single `data.frame` to be plotted

``` r
times$UK$COUNTRY <- "UK"
times$US$COUNTRY <- "US"
times <- do.call (rbind, times)
```

Bicycle journeys generally represent around one fifth of the proportion of walking journeys, so we multiply them by 5 to enable both to be plotted on the same scale

``` r
times$bike <- times$bike * 5
```

Now merge the variables into a single `country_trans_mode` variable for easy plotting:

``` r
require (tidyr)
times <- gather (times, key = trans_mode, value = time, walk, bike)
times$country_mode <- paste0 (times$COUNTRY, ":", times$trans_mode)
```

These data may now be used to plot changes in rates of walking and cycling over time, using the [`solarized` colours](http://ethanschoonover.com/solarized) from the [`ggthemes` package](https://github.com/jrnold/ggthemes).

``` r
require (ggplot2)
require (ggthemes)
ggplot (times, aes (year, time, colour = country_mode)) +
    geom_line () +
    geom_point () +
    theme_solarized ()
```

<img src="README-plot-1.png" width="672" />
