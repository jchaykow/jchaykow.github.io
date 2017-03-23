---
layout: post
title: "Fix City Typos in Springleaf Dataset"
author: mike_chaykowsky
modified:
excerpt: "Data Cleaning through Dplyr"
tags: []
---
In the spirit of data exploration I have another R code script here from the same Kaggle competition as my previous post Kaggle Springleaf.  The Website of the Kaggle competition is [Kaggle Springleaf](https://www.kaggle.com/c/springleaf-marketing-response) and you can get this code from the generosity of OttoP [here](https://www.kaggle.com/operdeck/springleaf-marketing-response/fix-up-city-names/run/75442).  If you go back and read the first post [*Kaggle Springleaf*](http://rcode.io/Kaggle-Springleaf/) you will see that the first set of features we explored were the character features. This code will deal with one specific character feature "VAR_0200" which we observed containing city names as character strings. OttoP realized that many of the city names had typos, so as a data cleaning exercise he fixed the names to be consistent.

### Goal: to fix the typos of the city names in one character feature.

All of the "library()" commands allow us to utilize these packages later on in the analysis.  We need to remember though that this cannot be run unless we first install.packages("") for each of these packages.  This can also be done by clicking on "tools" in the menu bar and then "Install Packages...".


{% highlight r %}
library(readr)
library(dplyr)
library(stringdist)
{% endhighlight %}

### Goal: read the dataset into R.

First we need to read in the training dataset.  Kaggle provides you with the dataset and you can access it at the link above.  Make sure that you first set your working directory (aka folder) to the folder that contains the dataset. Now we just want to find out how many unique character strings there are within the VAR_0200.  cat() does the same thing as concatenate, c(), but prints the output.  We can feed it character strings and numeric variables and it will print them all in the order we give it. So there are currently 12387 unique values.


{% highlight r %}
# Read competition data files:
train <- read_csv("train.csv")
{% endhighlight %}

{% highlight r %}
cat("Nr of city names before cleanup:", length(unique(train$VAR_0200)), fill=T)
{% endhighlight %}



{% highlight text %}
## Nr of city names before cleanup: 12387
{% endhighlight %}

### Goal: to create IDs that will uniquely identify similar City names.

So here is the meat of the coding and utilizes the dplyr package heavily for data cleaning.  I strongly recommend you checkout the ['dplyr cheat sheet'](https://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf) and just save it to your desktop for future use...very helpful.  Anyways, mutate() is a dplyr function that allows you to make new variables.  So we are just creating 3 new variables called 'City', 'State', and 'Zip', which are exactly the features from the original dataset.

%>% is called piping and is used frequently with dplyr.  It allows you to modify the same dataset as when you used mutate() but without having to call it again.  So now we are selecting those columns that we just created (City, State, Zip), so our new dataframe 'reviewDupes' will solely contain those columns.  Then we will use mutate() to create two new columns/variables, 'stateZip' and 'fullGeoID'. The idea of these two columns is they will act as a sort of ID value that we will use to match city names that are somewhat alike.  Say we have two rows, one has city "New York City" and stateZip "10006\_NY", and the other has city "NewYork City" and stateZip "10006\_NY".  In this case we will be able to group these rows according to their stateZip and then select the first city name to apply to both data. distinct() lets us only retain those rows from the original set that are unique. I added in the head(reviewDupes) so that we can get a better sense of what it looks like before we move on.


{% highlight r %}
reviewDupes <- mutate(train, City = VAR_0200, State = VAR_0237, Zip=VAR_0241) %>% 
  select(City, State, Zip) %>% 
  mutate(stateZip = paste(Zip, State, sep="_"), 
         fullGeoID = paste(City, Zip, State, sep="_")) %>% 
  distinct()
head(reviewDupes)
{% endhighlight %}



{% highlight text %}
## Source: local data frame [6 x 5]
## 
##            City State   Zip stateZip              fullGeoID
##           (chr) (chr) (int)    (chr)                  (chr)
## 1 FT LAUDERDALE    FL 33324 33324_FL FT LAUDERDALE_33324_FL
## 2        SANTEE    CA 92071 92071_CA        SANTEE_92071_CA
## 3    REEDSVILLE    WV 26547 26547_WV    REEDSVILLE_26547_WV
## 4       LIBERTY    TX 77575 77575_TX       LIBERTY_77575_TX
## 5     FRANKFORT    IL 60423 60423_IL     FRANKFORT_60423_IL
## 6        SPRING    TX 77379 77379_TX        SPRING_77379_TX
{% endhighlight %}

### Goal: create data.frame of grouped entries with the same stateZip IDs.

group\_by() takes the reviewDupes dataframe and subsets it so that all of the data with the same stateZip value are grouped into rows.  Hadley Wickham (the author of dplyr) refers to group\_by as an adverb.  group\_by determines what subsets of the data new functions will be applied to. Then the summarise() function collapses this to just the variable you grouped by and any arguments you pass it.  We will break this part up into pieces.  summarise(n = n(),...) condenses the dataframe by creating a dataframe with only two columns, the stateZip (that we used for group\_by a moment ago) and n (the number of 'stateZip's that were grouped for that case).  n() is a function that is only used from within the summarise() function, and possibly some other dplyr functions. Essentially n() is counting rows in each grouping after we just grouped the data. Then we create two more columns, altName and altID. Our new name is simply drawn from our groupings for each stateZip. We just grab the first city name in each grouping and use that as our new city name.  We do the same for altID. This is not a fool proof plan, but it cleans many of the entries as you will see. Now we only want to do this where we had cases to group, i.e. instances where there were multiple stateZip combos, so we filter for n greater than 1.


{% highlight r %}
potentialDupes <- group_by(reviewDupes, stateZip) %>% 
  dplyr::summarise(n = n(), 
                   altName = first(City), # prettier: most common
                   altID = first(fullGeoID)) %>% 
  filter(n > 1)
head(potentialDupes)
{% endhighlight %}



{% highlight text %}
## Source: local data frame [6 x 4]
## 
##   stateZip     n       altName                  altID
##      (chr) (int)         (chr)                  (chr)
## 1 10006_NY     2      NEW YORK      NEW YORK_10006_NY
## 2 10011_NY     2      NEW YORK      NEW YORK_10011_NY
## 3 10019_NY     2     MANHATTAN     MANHATTAN_10019_NY
## 4 10025_NY     2      NEW YORK      NEW YORK_10025_NY
## 5 10309_NY     2 STATEN ISLAND STATEN ISLAND_10309_NY
## 6 10463_NY     2         BRONX         BRONX_10463_NY
{% endhighlight %}

### Goal: place our fixed city names in one easily identifiable .csv file

If we use the dim() command, we can see that the reviewDupes data.frame is much larger than the potentialDupes data.frame.  This is just because the potentialDupes filtered for those City names that were grouped by stateZip and those that were not grouped were not included (this is the filter(n > 1) command).  Now we will combine the two sets we have created, potentialDupes and reviewDupes.  But we don't want everything from both. We use left\_join() by matching up the two data.frames using one column (stateZip) and then all of the extra columns from reviewDupes just get tacked on at the end of the data.frame along with the new column we create called 'dist'.  But keep in mind that if there is a stateZip in reviewDupes that is not in potentialDupes it will not be added to the new data.frame dupes. stringdist() is an interesting function in that it will tell you how far off one character string is from another.  If we look at the first element of dupes, altName = Elmont is dist = 1 from City = Elmony, because they differ by one character. In row 4, altName = Richmondhill is dist = 1 from City = Richmond Hill, because of the space.  Then we filter for only those entries that the stringdist is between 1 and 2, included.


{% highlight r %}
dupes <- mutate(left_join(potentialDupes, reviewDupes, by="stateZip"), 
                dist=stringdist(altName, City)) %>% 
                filter(dist >= 1 & dist <= 2)
head(dupes)
{% endhighlight %}



{% highlight text %}
## Source: local data frame [6 x 9]
## 
##   stateZip     n       altName                  altID            City
##      (chr) (int)         (chr)                  (chr)           (chr)
## 1 11003_NY     2        ELMONT        ELMONT_11003_NY          ELMONY
## 2 11103_NY     2        ASTORI        ASTORI_11103_NY         ASTORIA
## 3 11233_NY     2      BROOKLYN      BROOKLYN_11233_NY       BROOKLLYN
## 4 11418_NY     3  RICHMONDHILL  RICHMONDHILL_11418_NY   RICHMOND HILL
## 5 11419_NY     6 RICHMOND HILL RICHMOND HILL_11419_NY    RICMOND HILL
## 6 11419_NY     6 RICHMOND HILL RICHMOND HILL_11419_NY S RICHMOND HILL
## Variables not shown: State (chr), Zip (int), fullGeoID (chr), dist (dbl)
{% endhighlight %}



{% highlight r %}
write_csv(select(dupes, City, State, Zip, altName), "CleanedupCities.csv")
{% endhighlight %}

Here we can see some of the changes to city names.  It is not perfect, we can see in element 4 that it actually makes a correct entry incorrect.  However, in most cases it makes the correct change and with such a large feature in the dataset this is effective.


{% highlight r %}
print("Preview:")
{% endhighlight %}



{% highlight text %}
## [1] "Preview:"
{% endhighlight %}



{% highlight r %}
print(head(paste(dupes$City, dupes$State, "=>", dupes$altName), 20))
{% endhighlight %}



{% highlight text %}
##  [1] "ELMONY NY => ELMONT"                         
##  [2] "ASTORIA NY => ASTORI"                        
##  [3] "BROOKLLYN NY => BROOKLYN"                    
##  [4] "RICHMOND HILL NY => RICHMONDHILL"            
##  [5] "RICMOND HILL NY => RICHMOND HILL"            
##  [6] "S RICHMOND HILL NY => RICHMOND HILL"         
##  [7] "BAY SHORE NY => BAYSHORE"                    
##  [8] "DEER PARL NY => DEER PARK"                   
##  [9] "FARMINGVILLE NY => FARMINGVILE"              
## [10] "HOLTSVILLE NY => HOLTSCILLE"                 
## [11] "HUNTINGTON STATION NY => HUNTINGTION STATION"
## [12] "LEVITOWN NY => LEVITTOWN"                    
## [13] "NORTH PORT NY => NORTHPORT"                  
## [14] "GUILDERLAND NY => GUINDERLAND"               
## [15] "RENSSELAER NY => RENSESELEAR"                
## [16] "SOUTH GLENSFALLS NY => SOUTH GLENS FALLS"    
## [17] "SYRACUASE NY => SYRACUSE"                    
## [18] "NOERTH SYRACUSE NY => NORTH SYRACUSE"        
## [19] "WEST EDMESTON NY => WEST EDMINSTON"          
## [20] "BINGAHMTON NY => BINGHAMTON"
{% endhighlight %}

### Goal: Apply our changes to the train data.frame.

Now we will use the fullGeoID we created earlier to reform the original train data.frame.  First we create the fullGeoID in the train data.frame using mutate() and then pasting together the columns (i.e. ELMONY\_11003\_NY).  Then we just left\_join() the altName and fullGeoID columns of our dupes data.frame with train, organized by the fullGeoID using the select() function.  Then if we end up with an NA value anywhere we just place in the original City name that was in VAR\_0200.  ifelse() performs a logical test and then places in a value of our choice whether it is true or false.  So we ask it to check if there is an NA value in altName by calling is.na(altName), then if there is we place in the value from VAR\_0200 and if there isn't we place in our altName. Finally, we select all of the columns from train except for fullGeoID and altName.  To continue doing predictive analysis for this Kaggle we would have to perform this same process for the test data.frame.


{% highlight r %}
train <- mutate(train, fullGeoID = paste(VAR_0200, VAR_0241, VAR_0237, sep="_"))
train <- left_join(train, select(dupes, altName, fullGeoID), by="fullGeoID") %>%
  mutate(VAR_0200 = ifelse(is.na(altName), VAR_0200, altName)) %>%
  select(-fullGeoID, -altName)
# and do the same for the test set

cat("Nr of city names after cleansing:", length(unique(train$VAR_0200)), fill=T)
{% endhighlight %}



{% highlight text %}
## Nr of city names after cleansing: 10511
{% endhighlight %}
