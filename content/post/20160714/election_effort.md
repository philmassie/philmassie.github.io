---
title: "South African municipal elections 2016"
author: "Philip Massie"
date: 2016-07-14
draft: false
tags: [ "R", "D3", "Maps", "visualisation", "South Africa", "Election 2016" ]
output: 
  html_document: 
    keep_md: yes
    toc: no
---

### A visual comparison of party effort across municipalities / districts

With South Africa's municipal elections in a few days time (3rd August 2016), I wondered about the effort being expended by the parties in each district or municipality. Has it increased or decreased since the 2011 elections? I must point out that I am not a political analyst or a political scientist. I am nevertheless a scientist with an appreciation for analytics.

Employing and promoting candidates costs money. Assuming that our political parties don't have infinite financial resources it follows that investigating where they invest their resources may be a reasonable proxy for effort. Furthermore, looking at the change in effort adds a temporal dimension, suggesting where effort has increased or decreased between the elections.

The map below shows the change in PR candidate representation by each party across the county. If you hover your mouse pointer over a district, you can see by how much the representation has changed in terms of two metrics:

**Relative change**: The change in representation as a proportion of all representatives. E.g. In 2011 parties A and B each had 50% of a district's PR candidates. In 2016 party A has 60% and party B has 40%. Party A's relative change is 10% and party B's is -10%.

**Absolute change**: The change in actual number of PR candidates in a party from 2011 to 2016.

Scroll below the map for more info regarding data sources and methods.

*Disclaimer: As I said though, I am no political scientist and so maybe this is all nonsense.*



<iframe width="100%" height="800px" src="https://philmassie.github.io/election_effort_d3/za_election_effort_2016.html" scrolling = "no" frameborder="0" seamless name="iframe-election-content" id="iframe-election-content">
<p>Your browser does not support iframes. Click <a href="https://philmassie.github.io/election_effort_d3/">here</a> to navigate to the content.</p>
</iframe>

Embed this map using the following iframe code:

```
<iframe width="100%" height="660px" src="https://philmassie.github.io/election_effort_d3/za_election_effort_2016.html" scrolling = "no" frameborder="0" seamless name="iframe-election-content" id="iframe-election-content">
<p>Your browser does not support iframes. Click <a href="https://philmassie.github.io/election_effort_d3/">here</a> to navigate to the content.</p>
</iframe>
```

#### Data sources

The [Independent Electoral Commission]( http://www.elections.org.za) have published the [2016 candidate lists]( http://www.elections.org.za/content/Elections/Candidates-lists/) as pdf documents. The good people at [openAfrica](https://africaopendata.org/) have already gone to the trouble of [processing these pdfs](https://africaopendata.org/dataset/electoral-candidates-2016) (which can be a bit painful) so I used their lists instead. openAfrica also host the [2011](https://africaopendata.org/dataset/electoral-commission-of-south-africa-local-government-election-candidates) candidate lists. After learning a little more about the electoral system, it seemed that the proportional representative candidates would be the interesting group to look at.

The [Municipal Demarcation Board](http://www.demarcation.org.za/) website hosts shape files for South Africa's [districts](http://www.demarcation.org.za/index.php/downloads/boundary-data/boundary-data-main-files/districts) and [provinces](http://www.demarcation.org.za/index.php/downloads/boundary-data/boundary-data-main-files/province).

#### R data processing

Some simple wrangling in R (scroll down for the process) exposed the list of district codes which I joined with the appropriate candidate lists from each election year. From there I produced a list of parties, ordered by the total number of district or municipal representatives. The Economic Freedom Fighters lead this list with a total of 1501 candidates, 106 more than the African National Congress and 139 more than the Democratic Alliance. These are the three largest parties which is why I mention them. I chose to load the map with ANC data because the EFF didn't exist during the 2011 elections and so they've not decreased in any provinces. I believe that a few candidates have withdrawn from the elections. I'm not sure who they were but I doubt their withdrawal would make any substantive change to this analysis.



```r
## Libraries
library(rgdal) #
library(xlsx) #
library(plyr)#
library(stringr)#
library(dplyr) #
setwd("D:/My Folders/R/2016/blog/20160714_election_effort")
```

```r
# Load the district shape file to get a list of district codes
za.districts <- readOGR("Districts", layer="DistrictMunicipalities2011")
za.district.list <- unique(za.districts@data$DISTRICT)
rm(za.districts)
```

```r
# load and process electoral candidate data
# 2011 Proportional Representative candidates
pr.cand.2011 <- read.xlsx2("data/2011-pr-candidate-lists.xls", 1, startRow = 3)
# A few minor edits to party names
pr.cand.2011$Party <- str_to_title(str_replace_all(pr.cand.2011$Party, "  ", " "))
pr.cand.2011$Party[pr.cand.2011$Party == "Democratic Alliance/Demokratiese Alliansie"] <- "Democratic Alliance"
pr.cand.2011$Party[pr.cand.2011$Party == "Independent Ratepayers Association Of Sa"] <- "Independent Ratepayers Association Of SA"
pr.cand.2011$Party[pr.cand.2011$Party == "South African Maintanance And Estate Beneficiaries Associati"] <- "South African Maintanance And Estate Beneficiaries Association"
```

```r
# 2016 ward & proportional Representative candidates were in the same data set
ward.pr.cand.2016 <- read.csv("data/Electoral_Candidates_2016.csv")
# A few minor edits to party names
ward.pr.cand.2016$Party <- str_to_title(str_replace_all(ward.pr.cand.2016$Party, "  ", " "))
ward.pr.cand.2016$Party[ward.pr.cand.2016$Party == "Independent Ratepayers Association Of Sa"] <- "Independent Ratepayers Association Of SA"
ward.pr.cand.2016$Party[ward.pr.cand.2016$Party == "South African Maintanance And Estate Beneficiaries Associati"] <- "South African Maintanance And Estate Beneficiaries Association"

# A number of the 2016 district codes had a wierd "\f" prefix.
# I consulted the original IEC data,confirmed that these were errors, and tidied them up.
err_index <- grep("\f", ward.pr.cand.2016$Municipality)
ward.pr.cand.2016$Municipality[err_index] <- str_replace(ward.pr.cand.2016$Municipality[err_index], "\f", "")
ward.pr.cand.2016 <- droplevels(ward.pr.cand.2016)

#### Split the data into ward and pr data
# change variable names so they match later
pr.cand.2016 <- subset(ward.pr.cand.2016, ward.pr.cand.2016$PR.List.OrderNo...Ward.No < 1000)
names(pr.cand.2016)[4] <- "list.order.no"
```

```r
#### Process ward and PR data
processor <- function(df){
    # get the variable name of passed data
    data.name <- (deparse(substitute(df)))
    # split the municipality names into codes and names
    dist.split <- str_split_fixed(df$Municipality, " - ", 2)
    df$dist.code <- dist.split[,1]
    df$dist.name <- dist.split[,2]
    # Keep only the district values from the shape file. Main centers and DC areas.
    df <- df[df$dist.code %in% za.district.list, ]
    # split data by district and party, counting how many candidates per party per district
    df.long <- ddply(df, c("dist.code", "Party"), function(df) nrow(df))
    # temp naming - to keep track not NB
    names(df.long)[3] <- "num"
    # split new data by district, summing tot candidates from all parties (in district)
    df.tot <- ddply(df.long, "dist.code", function(df) sum(df$num))
    # join tot numbers by district to main data
    df.long <- join(df.long, df.tot, by = "dist.code")
    # temp naming - to keep track not NB
    names(df.long)[4] <- "tot"
    # calculate the proportion of each party to total, for each district
    df.long$prop <- df.long$num / df.long$tot
    # quick srting, not v NB
    df.long <- arrange(df.long, dist.code, -prop)
    # renaming based on variable name
    names(df.long)[c(3:5)] <- c(paste0(data.name, ".num"), paste0(data.name, ".tot"), paste0(data.name, ".prop"))
    return(df.long)
}
pr.cand.2011.long <- processor(pr.cand.2011)
pr.cand.2016.long <- processor(pr.cand.2016)

# Join data, dropping parties that no longer exist
pr.cand <- join(pr.cand.2011.long, pr.cand.2016.long, by = c("dist.code", "Party"), type = "right")

# replace NAs with zero for parties new in 2016
pr.cand[is.na(pr.cand)] <- 0

# calculate relative and absolute change
pr.cand$rel <- pr.cand$pr.cand.2016.prop - pr.cand$pr.cand.2011.prop
pr.cand$abs <- pr.cand$pr.cand.2016.num - pr.cand$pr.cand.2011.num
```

```r
# generate data for d3 map
# full data list
d3_data_all <- pr.cand[, c(1, 2, 9, 10)]
names(d3_data_all) <- c("dist_code", "party", "Relative Change", "Absolute Change")
write.csv(d3_data_all, file="data_d3/d3_data_all.csv", row.names = FALSE, quote = FALSE)

# Party data list, ordered by total number or district PR candidates
# group the data by Party
grouped <- group_by(pr.cand, Party)
# Summarise by summing the total candidates per party, excluding zeros and sorting
d3_party_list <- summarise(grouped, tot = sum(pr.cand.2016.num))
d3_party_list <- d3_party_list[d3_party_list$tot > 0, ]
d3_party_list <- arrange(d3_party_list, -tot)
# now we only need the party list
d3_party_list <- data.frame(party = d3_party_list$Party)

write.csv(d3_party_list, file="data_d3/d3_party_list.csv", row.names = FALSE, quote = FALSE)
```


The map was build with JavaScript and the mighty D3 libraries.
