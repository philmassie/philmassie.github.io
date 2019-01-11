---
title: "R Netcdf cheatsheet"
author: "Philip Massie"
date: 2016-08-04
draft: false
tags: [ "R", "NetCFD", "cheatsheet" ]
output: 
  html_document: 
    keep_md: yes
    toc: no

---

I recently promised a "NetCDF in R" cheat sheet to a friend, and I thought it might make a useful tutorial. NetCDF files are often used to distribute gridded, multidimensional spatial data such as sea surface temperature, chlorophyll-a levels and so on. NetCDF is more than just a file format, and so googling it can be a little intimidating. I hope this helps make these files a little easier to use in R.

A full specification for NetCDF can be found [here]( http://www.unidata.ucar.edu/software/netcdf/docs/). NetCDF files are really great for data distribution because they are self-describing, in other words they tell you what's inside. Additionally if you have a very large NetCDF data file, you can only pull out the subset of data you are interested in instead of opening the whole thing. There's more to them than this, but those are the important bits to me so far.

### Data

We'll use (amazingly) free chlorophyll-a data from [http://www.globcolour.info]( http://www.globcolour.info). These are gridded longitude by latitude values. Bear in mind that your NetCDF files may contain higher dimensions (e.g. longitude by latitude by time).

### R

I'm using [`ncdf4`](https://cran.r-project.org/web/packages/ncdf4/ncdf4.pdf) along with Hadley Wickham's [`reshape2`](https://cran.r-project.org/web/packages/reshape2/reshape2.pdf) and [`dplyr`](https://cran.r-project.org/web/packages/dplyr/dplyr.pdf) libraries.

```r
library(ncdf4)
library(reshape2)
library(dplyr)

# retrieve a list of nc files in my data folder:
flist <- list.files(path = "data/", pattern = "^.*\\.(nc|NC|Nc|Nc)$")
```
#### Investigate a NetCDF file

We'll dissect a single NetCDF file first and then we'll tie it all together in a function at the end.

```r
# Open a connection to the first file in our list
nc <- nc_open(paste0("data/", flist[1]))
```
You can get a description of what's inside the nc file using `print(nc)` which dumps info to your console. The output can be quite long so I prefer to save it to a text file. That way I can keep it open and avoid continuous scrolling in my console window.

```r
# Save the print(nc) dump to a text file (same name as the nc file with a txt extension)
{
    sink(paste0("data/", flist[1], ".txt"))
    print(nc)
    sink()
}
```
We can also access info about what's inside the NetCDF file by investigating its attributes. The term attributes might get a little confusing now.  I'll try to clarify which I'm referring to.

- We use R's `attributes()` function to get a list of a variable's attributes in the R workspace (`?attributes` for more). 

- The NetCDF data has its own list of global attributes (product info, spatial extents etc) as well as attributes for each NetCDF variable (units etc). 

Calling the R attributes of the NetCDF *file connection* provides access to some information about the file, e.g. variable names (var), number of variables (nvars) etc.

```r
# Get a list of the NetCDF's R attributes:
attributes(nc)$names
```

```
##  [1] "filename"    "writable"    "id"          "safemode"    "format"     
##  [6] "is_GMT"      "groups"      "fqgn2Rindex" "ndims"       "natts"      
## [11] "dim"         "unlimdimid"  "nvars"       "var"
```

```r
print(paste("The file has",nc$nvars,"variables,",nc$ndims,"dimensions and",nc$natts,"NetCDF attributes"))
```

```
## [1] "The file has 3 variables, 2 dimensions and 52 NetCDF attributes"
```
#### Retrieve the data

Calling the R attributes of "var" (revealed above), reveals the NetCDF variable names. These names in turn give us access to the data and associated NetCDF attributes (units etc).

Our data contains three grids, "CHL1_mean"  "CHL1_flags" "CHL1_error". We'll just focus on the chlorophyll-a data. The flags data is associated with cloud cover and the error describes error associated with each data point. These are fully described in the [Product User Guide](http://www.globcolour.info/CDR_Docs/GlobCOLOUR_PUG.pdf). We use the ncvar_get() function to save the data locally. refer to `?ncvar_get` for more on extracting subsets.

```r
# Get a list of the nc variable names.
attributes(nc$var)$names
```

```
## [1] "CHL1_mean"  "CHL1_flags" "CHL1_error"
```

```r
# Take a look at the chlorophyll variable's nc attributes (units etc).
ncatt_get(nc, attributes(nc$var)$names[1])
```

```
## $standard_name
## [1] "mass_concentration_of_chlorophyll_a_in_sea_water"
## 
## $long_name
## [1] "Chlorophyll concentration - Mean of the binned pixels"
## 
## $`_FillValue`
## [1] -999
## 
## $units
## [1] "mg/m3"
## 
## $pct_characterised_error
## [1] 43.31
```

```r
# Retrieve a matrix of the chlorophyll data using the ncvar_get function:
chla_mean <- ncvar_get(nc, attributes(nc$var)$names[1])

# Print the data's dimensions
dim(chla_mean)
```

```
## [1] 453 256
```
Now we will retrieve the latitude and longitude data stored as NetCDF dimensions ("dim"). We can compare the extents of these dimensions with those of our data matrix to confirm that they match. We will then use them to assign meaningful row and column names to our data.

```r
# Retrieve the latitude and longitude values.
attributes(nc$dim)$names
```

```
## [1] "lat" "lon"
```

```r
nc_lat <- ncvar_get( nc, attributes(nc$dim)$names[1])
nc_lon <- ncvar_get( nc, attributes(nc$dim)$names[2])

print(paste(dim(nc_lat), "latitudes and", dim(nc_lon), "longitudes"))
```

```
## [1] "256 latitudes and 453 longitudes"
```
These dimensions match those of the chl-a data which is arranged in a nc_lon(rows) by nc_lat(cols) matrix. This also matches what we saw in the text file. 

#### Tidy up a bit

We can make the data a little easier to think about with a bit of labelling and by transposing the data matrix so that the latitudes are in the rows and longitudes are in the columns. This isn't really critical though.

```r
# a quick look at a (pseudo) random section of this data
chla_mean[35:37, 245:247]
```

```
##           [,1]      [,2]      [,3]
## [1,] 0.1415056 0.1358830 0.1303250
## [2,] 0.1339336 0.1328560 0.1255906
## [3,] 0.1358152 0.1276533 0.1243571
```

```r
# Change the dimension names of our matrix to "lon" and "lat", 
# and the row and column names to the latitude and longitude values.
dimnames(chla_mean) <- list(lon=nc_lon, lat=nc_lat)
chla_mean[35:37, 245:247]
```

```
##                   lat
## lon                -36.4791717529297 -36.5208358764648 -36.5625038146973
##   16.6458396911621         0.1415056         0.1358830         0.1303250
##   16.6875057220459         0.1339336         0.1328560         0.1255906
##   16.7291717529297         0.1358152         0.1276533         0.1243571
```

```r
# lastly, you may want to transpose this matrix.
chla_mean <- t(chla_mean)
chla_mean[245:247, 35:37]
```

```
##                    lon
## lat                 16.6458396911621 16.6875057220459 16.7291717529297
##   -36.4791717529297        0.1415056        0.1339336        0.1358152
##   -36.5208358764648        0.1358830        0.1328560        0.1276533
##   -36.5625038146973        0.1303250        0.1255906        0.1243571
```
#### Global attributes

Let's take a look at the global NetCDF attributes. We saw above and in the text file that there were 52 global attributes in this file and they contain all kinds of useful info. Which attributes you will need is dependent on your analysis. We'll retrieve all the attributes with the `ncatt_get()` function. The `names()` function will give us a list of attribute names. These names will give us access to the relevant values. Thereafter it's easy to store whichever you want. I'll store the start and end date-times for this data set.

```r
# Retrieve the attributes
nc_atts <- ncatt_get(nc, 0)

# List all the attributes (commented to save space).
# names(nc_atts)

# Retrieve the start and end date-times
date_time_start <- as.POSIXct(nc_atts$start_time, format = "%Y%m%dT%H%M%SZ", tz = "UTC")
date_time_end <- as.POSIXct(nc_atts$end_time, format = "%Y%m%dT%H%M%SZ", tz = "UTC")
```
It's always a good idea to close the connection to the nc file when youre done.

```r
nc_close(nc)
```
#### Processing multiple NetCDF files

We've investigated various properties of a NetCDF file and seen how to extract and store variables. We have a whole directory of daily chlorophyll-a data files though and we can't process them one at a time. We'll write a small function to loop through them, and using what we've learned we will extract and add the data to a dataframe in long format. If you need other data structures it should be easy to adjust the function as required.

```r
# Define our function
process_nc <- function(files){
    # iterate through the nc
    for (i in 1:length(files)){
        # open a conneciton to the ith nc file
        nc_tmp <- nc_open(paste0("data/", files[i]))
        # store values from variables and atributes
        nc_chla <- ncvar_get(nc_tmp, attributes(nc_tmp$var)$names[1])
        nc_lat <- ncvar_get(nc_tmp, attributes(nc_tmp$dim)$names[1])
        nc_lon <- ncvar_get(nc_tmp, attributes(nc_tmp$dim)$names[2])
        nc_atts <- ncatt_get(nc_tmp, 0)
        nc_start_date <- as.Date(nc_atts$period_start_day, format = "%Y%m%d", tz = "UTC")
        # close the connection sice were finished
        nc_close(nc_tmp)
        # set the dimension names and values of your matrix to the appropriate latitude and longitude values
        dimnames(nc_chla) <- list(lon=nc_lon, lat=nc_lat)

        # I'm choosing to store all the data in long format.
        # depending on your workflow you can make different choices here...
        # Your variable may get unmanageably large here
        # if you have high spatial and temporal resolution nc data.
        tmp_chl_df <- melt(nc_chla, value.name = "chla")
        tmp_chl_df$date_start <- nc_start_date

        # set the name of my new variable and bind the new data to it
        if (exists("chla_data_monthly")){
            chla_data_monthly <- bind_rows(chla_data_monthly, tmp_chl_df)
        }else{
            chla_data_monthly <- tmp_chl_df
        }
        # tidy up, not sure if necesarry really, but neater
        rm(nc_chla, nc_lat, nc_lon, nc_tmp, nc_atts, nc_start_date, tmp_chl_df)
    }

    return(chla_data_monthly)
}


data <- process_nc(flist)
```
Ok that's it. I hope someone finds this useful :).
