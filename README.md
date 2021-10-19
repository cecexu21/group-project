# group-project

---
title: "Final Group Project: AirBnB analytics"
date: "12 Oct 2021"
author: "Reading Time: About 8 minutes"
output:
  html_document:
    highlight: zenburn
    theme: flatly
    toc: yes
    toc_float: yes
    number_sections: yes
    code_folding: show
---


```{r setup, include=FALSE}
options(knitr.table.format = "html") 
knitr::opts_chunk$set(warning = FALSE, message = FALSE, 
  comment = NA, dpi = 300)
```


```{r load-libraries, echo=FALSE}
library(corrplot)
library(tidyverse) 
library(car) 
library(lubridate) 
library(GGally) 
library(ggfortify) 
library(rsample) 
library(janitor) 
library(broom) 
library(huxtable) 
library(kableExtra) 
library(moderndive) 
library(skimr) 
library(mosaic)
library(leaflet) 
library(tidytext)
library(viridis)
library(stringr)
library(vroom)
library(psych)
```




In our final group assignment we will analyse data about Airbnb listings and fit a model to predict the total cost for two people staying 4 nights in an AirBnB in a city. 


```{r load_data, echo=FALSE, message=FALSE, warning=FALSE, cache=TRUE, results=FALSE}
listings <- vroom("http://data.insideairbnb.com/australia/nsw/sydney/2021-09-08/data/listings.csv.gz") %>% 
       clean_names()

glimpse(listings)

#Let's give a first look at the variables we have in the dataset. In order to perform our analysis we will consider some of them: in this sense, among the most relevant ones, we can highlight price, accomodates, minimum nights and maximum nights, review ratings score, review ratings cleaniliness, review ratings value and review ratings communication as well as the reviews per months.

listings %>%
  group_by(host_location) %>% #looking at designations of host locations on the website
  summarise(number_of_location=n()) %>% #n() is equivalent to count
  arrange(desc(number_of_location)) 

```



```{r skimming, cache=TRUE}
#By utilizing the skim function, it is possible to define a schematic organization of our data and identify some crucial elements. It is possible to notice, first of all, that as far as the variable price is concerned, the dataset does not miss any value and the same is also true for the property type. The mean values for beds and bedrooms are both around 2 while bathrooms statistics are not available since their data are missing. The number of nights spent is between 6 and 8 and on avergae 3 people live in the house.
skim(listings)
```




```{r basic stats, cache=TRUE}
favstats(~number_of_reviews, data = listings)

# there are on average 15 reviews per house but with notable outliers since the median number is 2.

favstats(~reviews_per_month, data = listings)

# per month there are almost 0.5 reviews with values that can shift from 0 to 54 per month.

favstats(~review_scores_rating, data = listings)

# this statistic is crucial and very informative: it results quite unbiased with a median of 4.82 and a mean of 4.41 (values are included between 0 and 5) computed from more than 10000 reviews.

favstats(~bathrooms, data = listings)

#it is important to highlight that the statistic about bathrooms is not available because of missing values.

favstats(~review_scores_cleanliness, data = listings)

# as in the case of ratings, cleanliness results to be unbiased with several observations (more than 10000) and shows a median of 4.81 and a mean of 4.58 (with values between 0 and 5).

favstats(~review_scores_communication, data = listings)

# communication too appears to be unbiased and reliable with more than 10000 observations.

favstats(~review_scores_checkin, data = listings)

favstats(~review_scores_location, data = listings)

favstats(~maximum_nights, data = listings)
favstats(~minimum_nights, data = listings)

# no missing data for both variables, with more than 30000 observations. Standard deviations appear to be very high.
```


  
```{r location ratings, cache=TRUE}
ggplot(data=listings, aes(x=review_scores_rating , y=review_scores_cleanliness  , group=1)) +
  geom_point()+
  ggtitle("Relationship between ratings and cleanliness scores") +
  xlab("Ratings") + ylab("Cleanliness")

#we tested the relationship between ratings and cleaniliness scores in order to understand how changes in the feeling of cleaniliness affect the overall rating score.

ggplot(data=listings, aes(x=host_identity_verified )) + 
  geom_bar(color="black", fill="white")+
   ggtitle("Number of verified hosts per listing")+ 
  xlab("Verified hosts") + ylab("Number of Listings")

# We analyzed the values regarding the verified hosts to understand whether AirBNB considers verified hosts.


ggplot(data=listings, aes (x= review_scores_location ))+
  geom_histogram()+
  stat_bin(bins=30)+ 
   ggtitle("Location ratings") +
  xlab("Ratings") + ylab("Number of Listings")

# We considered also the scores on the basis of the different locations in Sydney
```
  
        
```{r listings price, cache=TRUE}
skim(listings)


listings <- listings %>% 
  mutate(price = parse_number(as.character(price))) #converting price from char to double

typeof(listings$price)
```

```{r corrplot, cache=TRUE}
listings_num_variables <- listings %>% 
  select(where(is.numeric))

dropping<- c("host_total_listings_count", "price", "scrape_id", "host_id", "latitude", "longitude", "minimum_minimum_night", "minimum_maximum_night", "maximum_minimum_night", "maximum_minimum_night", "availability_30",  "availability_60 ", "availability_90","availability_365","calculated_host_listings_count","calculated_host_listings_count_entire_homes","calculated_host_listings_count_private_rooms", "calculated_host_listings_count_shared_rooms", "minimum_minimum_nights","maximum_minimum_nights", "minimum_maximum_nights","maximum_maximum_nights", "minimum_nights_avg_ntm", "maximum_nights_avg_ntm")

listings_num_variables<- listings_num_variables[, !(names(listings_num_variables)%in% dropping)]  #Removed deplicated variables

listings_num_variables <- listings_num_variables %>% 
  filter(maximum_nights< 360) 

listings_num_variables

library(corrplot)
correl = cor(listings_num_variables, use="pairwise.complete.obs")
corrplot(correl, method = "circle") #let's look at the correlation in a different form
```


```{r ggpairs, cache=TRUE}
dropping2<- c("beds", "number_of_reviews_ltm","number_of_reviews_l30d","review_scores_accuracy", "review_scores_checkin", "review_scores_location", "review_scores_value", "review_per_month", "host_listings_count")

listings_num_variables<- listings_num_variables[, !(names(listings_num_variables)%in% dropping2)]  #Removed other less relevant variables
listings_num_variables

ggpairs(listings_num_variables) #Analyzing the correlations between the variables

skim(listings_num_variables)

```



From the initial dataset we have tried to eliminate all repetitive or less useful variables. We have initially statistically analyze the dataset to comprehend the situation at the beginning by utilizing skrim and listings. Later on, we modified the dataset and we start analyzing potential correlations which results from the graphs. Specifically, it appears that correlations do not result to be linear.


- How many variables/columns? How many rows/observations? 

We had 74 variables at the beginning, we achieved a final value of 11 after the different drops. We had 31030 observations at the beginnig, we achieved a final value of 12314 after modifying the dataset.


- Which variables are numbers?  

At the beginning they were: id, scrape_id, host_id, host_listings_count, host_total_listings_count, latitude, longitude, accommodates, bedrooms,beds, minimum_nights, maximum_nights, minimum_minimum_nights, maximum_minimum_nights, minimum_maximum_nights, maximum_maximum_nights, minimum_nights_avg_ntm, maximum_nights_avg_ntm, availability_30, availability_60, availability_90, availability_365, number_of_reviews, number_of_reviews_ltm, number_of_reviews_l30d, review_scores_rating review_scores_accuracy, review_scores_cleanliness, review_scores_checkin, review_scores_communication, review_scores_location, review_scores_value, calculated_host_listings_count, calculated_host_listings_count_entire_homes, calculated_host_listings_count_private_rooms, calculated_host_listings_count_shared_rooms, reviews_per_month. 


After modifying the dataset: id, accommodates, bedrooms, minimum_nights, maximum_nights, availability_60, number_of_reviews, review_scores_rating, review_scores_cleanliness, review_scores_communication, reviews_per_month. 
- Which are categorical or *factor* variables (numeric or character variables with variables that have a fixed and known set of possible values? 

listing_url, name, description, neighborhood_overview, picture_url, host_url, host_name, host_location, host_about, host_response_time, host_response_rate, host_acceptance_rate, host_thumbnail_url, host_picture_url, host_neighbourhood, host_verifications, neighbourhood, neighbourhood_cleansed, property_type, room_type, bathrooms_text, amenities,price,license.


- What are the correlations between variables? Does each scatterplot support a linear relationship between variables? Do any of the correlations appear to be conditional on the value of a categorical variable?

The correlations demonstrate that the variables are not linearly related.Outside of the same typology of class (the different types of reviews for instance), we have correlation coefficients that are quite low. This is because many variables are not linearly correlated even though.

## Data wrangling


```{r listings price 2, cache=TRUE}
listings <- listings %>% 
  mutate(price = parse_number(as.character(price))) #converting price from character to double

typeof(listings$price)

```

  
Used `typeof(listing$price)` to confirm that `price` is now stored as a number.


## Propery types


Next, we look at the variable `property_type`. We can use the `count` function to determine how many categories there are their frequency. What are the top 4 most common property types? What proportion of the total listings do they make up? 


```{r listings category,cache=TRUE}
listings_category <- listings %>% #find the rankings of each category of property type
  group_by(property_type) %>%
  summarise(number_of_category=count(property_type)) %>% #counting each property type
  arrange(desc(number_of_category)) %>% # descending
  mutate(all_category=sum(number_of_category),category_proportion=number_of_category/all_category) %>% #property type as proportion of total number of categories
  head(4) #finding the top 4 most common property types

listings_category

listings_category_top_4 <- listings_category %>% #what proportion of all properties are made up by our top 4?
  summarise(top_four_category=sum(category_proportion))

listings_category_top_4
```

Since the vast majority of the observations in the data are one of the top four or five property types, we would like to create a simplified version of `property_type` variable that has 5 categories: the top four categories and `Other`. Fill in the code below to create `prop_type_simplified`.

Use the code below to check that `prop_type_simplified` was correctly made.

```{r property type, cache=TRUE}
listings <- listings %>%
  mutate(prop_type_simplified = case_when(
    property_type %in% c("Entire rental unit","Private room in rental unit", "Entire residential home","Private room in residential home") ~ property_type, 
    TRUE ~ "Other"
  ))
```
  
```{r arrange property type, cache=TRUE}
listings %>% #the above new column displayed and compared with old classification
  count(property_type, prop_type_simplified) %>%
  arrange(desc(n))    
```



Airbnb is most commonly used for travel purposes, i.e., as an alternative to traditional hotels. We only want to include listings in our regression analysis that are intended for travel purposes:

- What are the most common values for the variable `minimum_nights`? 

```{r listings minimum nights, cache=TRUE}
listings %>% #by taking a count, we can figure out which values are most common
  count(minimum_nights) %>%
  arrange(desc(n))
```

- Is there any value among the common values that stands out? What is the likely intended purpose for Airbnb listings with this seemingly unusual value for `minimum_nights`?

7 days minimum stay is more common than 4,5 and 6 days, which stands out as it is for a longer period of time. but this can be justified since it is likely that rentors would like to have their properties in use for a week at a time rather than have the renting period finish randomly midweek.

# Mapping 

The following code, having downloaded a dataframe `listings` with all AirbnB listings in Sydney, will plot on the map all AirBnBs where `minimum_nights` is less than equal to four (4).


```{r leaflet, out.width = '80%', cache=TRUE}
leaflet(data = filter(listings, minimum_nights <= 4)) %>% #Using leaflet to display a map of the properties in our dataframe with minimum nights fewer than 4
  addProviderTiles("OpenStreetMap.Mapnik") %>% 
  addCircleMarkers(lng = ~longitude, 
                   lat = ~latitude, 
                   radius = 1, 
                   fillColor = "blue", 
                   fillOpacity = 0.4, 
                   popup = ~listing_url,
                   label = ~property_type)
```

    
# Regression Analysis

For the target variable $Y$, we will use the cost for two people to stay at an Airbnb location for four (4) nights. 

We will create a new variable called `price_4_nights` that uses `price`, and `accomodates` to calculate the total cost for two people to stay at the Airbnb property for 4 nights. This is the variable $Y$ we want to explain.

```{r 4 nights, cache=TRUE}
# in this part i delete neighbourhood_group_cleaned because it will be used in the analysis part(kostis asked us to do) and license(since it may have some impact in the final model)
drop_columns <- (c("id", #useless in our analysis
                 "listing_url", #useless in our analysis
                 "scrape_id", #useless in our analysis
                 "last_scraped", #useless in our analysis
                 "name", #useless in our analysis
                 "description", #useless in our analysis
                 "neighborhood_overview", #useless in our analysis
                 "picture_url", #useless in our analysis
                 "host_id", #useless in our analysis
                 "host_url", #useless in our analysis
                 "host_name", #useless in our analysis
                 "host_about", #useless in our analysis
                 "host_thumbnail_url", #useless in our analysis
                 "host_picture_url", #useless in our analysis
                 "bathrooms", #contains only NAs
                 "minimum_minimum_nights", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "maximum_minimum_nights", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "minimum_maximum_nights", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "maximum_maximum_nights", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "minimum_nights_avg_ntm", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "maximum_nights_avg_ntm", #inconsistent data, removed following this advice: https://medium.com/@kalenderselmir/munichs-airbnb-data-analysis-fd815f2c918f
                 "calendar_updated", #contains only NAs
                 "calendar_last_scraped",
                 "first_review",                                
                 "last_review",
                 "calendar_updated",
                 "calculated_host_listings_count",
                 "calculated_host_listings_count_entire_homes",
                 "calculated_host_listings_count_private_rooms",
                 "calculated_host_listings_count_shared_rooms"))

listings_sydney <- listings %>% # creating a new dataframe without the useless columns, keeping our old df intact
  select(-drop_columns)


bathrooms_list<-unique(as.character(listings_sydney$bathrooms_text))

listings_sydney_2 <- listings_sydney %>% # we withdraw the numbers from the below strings
  mutate(bathrooms_number=case_when(bathrooms_text=="1 shared bath"~1,
                                    bathrooms_text=="3 baths"~3,
                                    bathrooms_text=="1 private bath"~1,
                                    bathrooms_text=="1 bath"~1,
                                    bathrooms_text=="1.5 shared baths"~1.5,
                                    bathrooms_text=="2.5 shared baths"~2.5,
                                    bathrooms_text=="2 baths"~2,
                                    bathrooms_text=="1.5 baths"~1.5,
                                    bathrooms_text=="2.5 baths"~2.5,
                                    bathrooms_text=="0 baths"~0,
                                    bathrooms_text=="2 shared baths"~2,
                                    bathrooms_text=="4 baths"~4,
                                    bathrooms_text=="3 shared baths"~3,
                                    bathrooms_text=="Half-bath"~0.5,
                                    bathrooms_text=="Shared half-bath"~0.5,
                                    bathrooms_text=="3.5 baths"~3.5,
                                    bathrooms_text=="3.5 shared baths"~3.5,
                                    bathrooms_text=="5 baths"~5,
                                    bathrooms_text=="4.5 baths"~4.5,
                                    bathrooms_text=="0 shared baths"~0,
                                    bathrooms_text=="6 baths"~6,
                                    bathrooms_text=="5.5 bathss"~5.5,
                                    bathrooms_text=="6 shared bath"~6,
                                    bathrooms_text=="Private half-bath"~0.5,
                                    bathrooms_text=="8 baths"~8,
                                    bathrooms_text=="4 shared baths"~4,
                                    bathrooms_text=="7 baths"~7,
                                    bathrooms_text=="6.5 baths"~6.5,
                                    bathrooms_text=="5.5 shared baths"~5.5,
                                    bathrooms_text=="4.5 shared baths"~4.5,
                                    bathrooms_text=="5 shared bathss"~5,
                                    bathrooms_text=="14.5 shared baths"~14.5,
                                    bathrooms_text=="7 shared baths"~7,
                                    bathrooms_text=="10 baths"~10))

#coordinates for Sydney Opera house: latitude -33.8568°, longitude  151.2153°
#forumla for distance between two coordinates: sqrt((x1-x2)^2+(y1-y2)^2)
listings_sydney_opera_distance <- listings_sydney_2 %>% 
  mutate(distance_opera=sqrt((latitude-(-33.8568))^2+(longitude-151.2153)^2))

listings_sydney_golden <- listings_sydney_opera_distance %>% #we now create our 'Golden' dataframe that we will use for the rest of our analysis
  filter(grepl("Sydney",host_location)) %>% #filter for location name to include "Sydney"
  filter(accommodates>=2, #to find price for 4 nights for 2 people first we restrict to those properties that can accommodate 2 or more people
         minimum_nights<=4) %>% #we can't consider properties that require you to stay for more than 4 nights and hence filter them out
  mutate(price_4_nights=price*4) %>% #we now take the price per night for the rooms that satisfy the above and multiply by 4 to get `price_4_nights`
  arrange(desc(price_4_nights)) %>% # note that we aren't calculating pro rata and multiplying by two as we are assuming that if 2 people book a property for 4 people, they still have to pay full price
  mutate(log_price_4_nights=log(price_4_nights))

```

We now use histograms and density plots to examine the distributions of `price_4_nights` and `log_price_4_nights`. Which variable should you use for the regression model? Why?

```{r log price 4 nights, cache=TRUE}
ggplot(listings_sydney_golden,aes(price_4_nights))+ #price_4_nights is a very right skewed data set
  geom_density(aes(x=price_4_nights))

ggplot(listings_sydney_golden,aes(price_4_nights))+ #...which leads to a very right skewed histogram 
  geom_histogram(aes(x=price_4_nights), bins=50)

ggplot(listings_sydney_golden,aes(log_price_4_nights))+ #log_price_4_nights is still right skewed but a lot less so
  geom_density(aes(x=log_price_4_nights))

ggplot(listings_sydney_golden,aes(log_price_4_nights))+ #... further highlighted by this histogram showing a close to normal distribution
  geom_histogram(aes(x=log_price_4_nights),bins=30)
```


We will fit a regression model called `model1` with the following explanatory variables: `prop_type_simplified`, `number_of_reviews`, and `review_scores_rating`. 

```{r pairs panel, cache=TRUE}

model1 <-lm(log_price_4_nights ~  prop_type_simplified+number_of_reviews+review_scores_rating, data = listings_sydney_golden)
model1 %>% 
  glance()
msummary(model1) 
pairs.panels(listings_sydney_golden[c("prop_type_simplified","number_of_reviews","review_scores_rating")])
autoplot(model1)+theme_bw()

```


- Interpret the coefficient `review_scores_rating` in terms of `log_price_4_nights`.

The coefficient is statistically significant and represents a 2.6% change in our Y for every 1 increase in rating.

- Interpret the coefficient of `prop_type_simplified` in terms of `log_price_4_nights`.

The coefficients of each property type is significant. Each category can only take a value of 0 or 1, depending on if the property is in the category or not. generally, renting an "Entire residential home" will lead to an increase in price, whilst renting "Private room in rental unit", "Private room in residential home", or any other type of property will lead to a decrease in price- all highlighted by the sign of the coefficients.


We want to determine if `room_type` is a significant predictor of the cost for 4 nights, given everything else in the model. Fit a regression model called model2 that includes all of the explanatory variables in `model1` plus `room_type`. 
```{r msummary, cache=TRUE}

model2 <-lm(log_price_4_nights ~  prop_type_simplified+number_of_reviews+review_scores_rating+room_type, data = listings_sydney_golden)
msummary(model2)
pairs.panels(listings_sydney_golden[c("prop_type_simplified","number_of_reviews","review_scores_rating","room_type")])

```
This model seems worse than the one prior despite an increased R squared since we can see that "prop_type_simplifiedPrivate room in rental unit" and "room_typeHotel room" are both insignificant.


## Further variables/questions to explore on our own

Our dataset has many more variables.

1. Are the number of `bathrooms`, `bedrooms`, `beds`, or size of the house (`accomodates`) significant predictors of `price_4_nights`? Or might these be co-linear variables?

```{r bathrooms, cache=TRUE}
#we will have more analysis on new variables, so pervious variables should also be included

model_bathrooms <-lm(log_price_4_nights ~  bathrooms_number, data = listings_sydney_golden)
msummary(model_bathrooms)
```


It seems as though it is!

```{r bedrooms, cache=TRUE}
#bedrooms
listings_sydney_bedrooms <- listings_sydney_golden
 #replacing NA values in bedrooms - using base R as recode is not working (we cannot use this way to change original number)

model_bedrooms <-lm(log_price_4_nights ~  bedrooms, data = listings_sydney_bedrooms)
msummary(model_bedrooms)

```

`bedrooms` works too!

```{r beds, cache=TRUE}
#beds
listings_sydney_beds <- listings_sydney_golden
 #replacing NA values in bedrooms - using base R as recode is not working (we cannot use this method to change any original data)

model_beds <-lm(log_price_4_nights ~  beds, data = listings_sydney_beds)
msummary(model_beds)

```

`beds` works as well!

```{r accommodates, cache=TRUE}
model_accommodates <-lm(log_price_4_nights ~  accommodates, data = listings_sydney_golden)
msummary(model_accommodates)
```

As expected, `accommodates` is also significant.

Now what happens when we put the above altogether?

```{r co_linearity_between_4_variables, cache=TRUE}
#test colinearity between bathrooms_number, bedrooms, bdes and accommodates
model_4_variables <-lm(log_price_4_nights~ bathrooms_number+bedrooms+beds+accommodates, data=listings_sydney_golden)
msummary(model_4_variables)
car::vif(model_4_variables) #generally speaking, we should keep variables with vif ranging from 1 to 10, so all these four variables can be kept
autoplot(model_4_variables)+theme_bw()

```

Each variable together gives us a better model than any of the prior but our vif>5 for accommodates


```{r beds_bathroom_accommodates_previous_variables, cache=TRUE}
model_4_variables_final <-lm(log_price_4_nights~ bathrooms_number+bedrooms+beds+accommodates+prop_type_simplified+number_of_reviews+review_scores_rating+room_type, data=listings_sydney_golden)
msummary(model_4_variables_final)
car::vif(model_4_variables_final) #generally speaking, we should keep variables with vif ranging from 1 to 5, so all these four variables can be kept, note that prop_type_simplified, room_type and bedrooms all have vif is greater than 5, so we remove prop_type_simplified and hope we can receive a reduced vif for the latter two in a future model.

#more reasons such as why this happens
autoplot(model_4_variables_final)+theme_bw()

model_4_variables_final2 <-lm(log_price_4_nights~ bathrooms_number+bedrooms+beds+accommodates+number_of_reviews+review_scores_rating+room_type, data=listings_sydney_golden)
msummary(model_4_variables_final2)
car::vif(model_4_variables_final2) 
autoplot(model_4_variables_final2)+theme_bw()
#note that accommodates' vif is greater than 5
```


```{r distance_from_opera_house, cache=TRUE}

#coordinates for Sydney Opera house: latitude -33.8568°, longitude  151.2153°
#forumla for distance between two coordinates: sqrt((x1-x2)^2+(y1-y2)^2)

model_opera <-lm(log_price_4_nights ~  distance_opera, data = listings_sydney_golden)
summary(model_opera)

```

1. Do superhosts `(host_is_superhost`) command a pricing premium, after controlling for other variables?

```{r superhosts, cache=TRUE}

#categorical variables- superhost to show whether it has a pricing premium

model_superhost<-lm(log_price_4_nights~ bathrooms_number+bedrooms+beds+accommodates+host_is_superhost+number_of_reviews+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_superhost)
car::vif(model_superhost)
# vif for accommodates is 6.06
```

1. Some hosts allow you to immediately book their listing (`instant_bookable == TRUE`), while a non-trivial proportion don't. After controlling for other variables, is `instant_bookable` a significant predictor of `price_4_nights`?

```{r instant_bookable, cache=TRUE}
model_instant_bookable<- lm(log_price_4_nights~bathrooms_number+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+number_of_reviews+review_scores_rating+room_type, data=listings_sydney_golden)

msummary(model_instant_bookable)
car::vif(model_instant_bookable)
autoplot(model_instant_bookable)+theme_bw()
```

1. For all cities, there are 3 variables that relate to neighbourhoods: `neighbourhood`, `neighbourhood_cleansed`, and `neighbourhood_group_cleansed`. There are typically more than 20 neighbourhoods in each city, and it wouldn't make sense to include them all in your model. Use your city knowledge, or ask someone with city knowledge, and see whether you can group neighbourhoods together so the majority of listings falls in fewer (5-6 max) geographical areas. You would thus need to create a new categorical variabale `neighbourhood_simplified` and determine whether location is a predictor of `price_4_nights`

```{r neighbourhood_cleansed, cache=TRUE}
listings_sydney_golden %>%
  group_by(neighbourhood_cleansed)%>%
  summarise(count=n()) %>%
  arrange(desc(count)) 

# since we have already chosen great sydney as our target city, we will divide neighbourhoods based on their geographic locations into 5 parts-central sydney, east sydney, north sydney, west sydney and south sydeny
neighbourhood_location<-c("central sydney","east sydney","north sydney","west sydney","south sydney")
                          
listings_sydney_golden<-listings_sydney_golden %>%
  mutate(neighbourhood_simplified=case_when(
    neighbourhood_cleansed %in% c("Sydney") ~ "Central",
    neighbourhood_cleansed %in% c("Botany Bay","Camden","Waverley","Randwick","Woollahra") ~ "East",
    neighbourhood_cleansed %in% c("North Sydney","Warringah","Manly","Pittwater","Mosman","Hornsby","Ku-Ring-Gai","Lane Cove","Hunters Hill","Willoughby") ~ "North",
    neighbourhood_cleansed %in% c("Rockdale","Sutherland Shire","Hurstville","City of Kogarah") ~ "South",
    neighbourhood_cleansed %in% c("Marrickville","Leichhardt","Ryde","Auburn","Canada Bay","Parramatta","Canterbury","Burwood","Ashfield","Blacktown","Bankstown","The Hills Shire","Strathfield","Penrith","Fairfield","Campbelltown","Liverpool")~ "West", 
    TRUE ~ "Other"))

model_neighbourhood<-lm(log_price_4_nights~neighbourhood_simplified,data=listings_sydney_golden)
msummary(model_neighbourhood)

# locations:significant,but not significant in East Sydney. Why? Maybe because of the economic status of that part? find more PEST factors(needs more interpretation)

# then testing whether 'neighbourhood_simplified' is a significant predictor for price_4_nights by controlling other variables
model_neighbourhood_cleansed<-lm(log_price_4_nights~neighbourhood_simplified+bathrooms_number+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+number_of_reviews+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_neighbourhood_cleansed)
car::vif(model_neighbourhood_cleansed)
# with F statistic's p-value smaller than 0.001, the model itself is significant, and adjusted R-square is getting greater
autoplot(model_neighbourhood_cleansed)+theme_bw()

model_neighbourhood_cleansed2<-lm(log_price_4_nights~neighbourhood_simplified+bathrooms_number+bedrooms+accommodates+host_is_superhost+instant_bookable+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_neighbourhood_cleansed2)
car::vif(model_neighbourhood_cleansed2)

#anova part is to figure out whether neighbourhood_cleansed has impact on model, since F statistic is large enough and p-value is smaller than 0.001, this variable should be kept
anova(model_instant_bookable,model_neighbourhood_cleansed)

```


```{r immediate_booking, cache=TRUE}

#not sure what it is for, in order to control other variables, it should first run a linear regression model without this variable, but containing other variables(like x1,x2 ...)and then run a new linear regress model containing all variables 
model_immediate_booking<-lm(log_price_4_nights~has_availability,data=listings_sydney_golden)
msummary(model_immediate_booking)
# the factor is not significant on alpha=0.001, so it means we should drop this variable in new models

#after adding other variables, it turns out that there is no pricing premium with the variable of model_immediate_booking, since the adjusted R-squared is still 0.447 and the p-value for has_availability is greater than 0.01, so we decided to drop this variable since it cannot be a useful variable for predictions
model_immediate_booking_final<-lm(log_price_4_nights~neighbourhood_simplified+bathrooms_number+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+number_of_reviews+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_immediate_booking_final)
car::vif(model_immediate_booking_final)
autoplot(model_immediate_booking_final)+theme_bw()


```

1. What is the effect of `availability_30` or `reviews_per_month` on `price_4_nights`, after we control for other variables?

```{r availability_30, cache=TRUE}
model_availability<-lm(log_price_4_nights~availability_30+neighbourhood_simplified+bathrooms_number+bedrooms+beds+accommodates+host_is_superhost+instant_bookable+number_of_reviews+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_availability)
car::vif(model_availability)
autoplot(model_availability)+theme_bw()

#this time we found that host_is_superhost, beds and number of reviews are not significant any more, and adjusted R-squared is much greater than previous models, so considering about whether we need to drop these variable

model_availability<-lm(log_price_4_nights~availability_30+neighbourhood_simplified+bathrooms_number+bedrooms+accommodates+instant_bookable+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_availability)
car::vif(model_availability)
autoplot(model_availability)+theme_bw()


```
```{r reviews_per_month, cache=TRUE}
# with this model, we found that host_is_superhost is significant again, but reviews_per_month is not significant.
model_reviews<-lm(log_price_4_nights~reviews_per_month+neighbourhood_simplified+bathrooms_number+bedrooms+accommodates+host_is_superhost+instant_bookable+review_scores_rating+room_type,data=listings_sydney_golden)

msummary(model_reviews)
car::vif(model_reviews)

# then run a model with all other variables we used before as instructions and 'reviews_per_month','availability_30' and host is superhost is siginificant again
model_reviews_availability<-lm(log_price_4_nights~availability_30+reviews_per_month+neighbourhood_simplified+bathrooms_number+bedrooms+accommodates+host_is_superhost+instant_bookable+review_scores_rating+room_type,data=listings_sydney_golden)
msummary(model_reviews_availability)
car::vif(model_reviews_availability)
autoplot(model_reviews_availability)+theme_bw()

#anova part is to figure out whether number of reviews has impact on model, since F statistic is large enough and p-value is smaller than 0.001, this variable should be kept
anova(model_availability,model_reviews_availability)


```

## Diagnostics, collinearity, summary tables



```{r ggpairs matrix, cache=TRUE}
listings_sydney_golden1 <- listings_sydney_golden %>% 
  select(log_price_4_nights,bathrooms_number, distance_opera,host_since) %>% 
  ggpairs()

# first, we uses all variables mentioned above and create a residual plot
listings_sydney_golden2 <- listings_sydney_golden %>% 
  select(log_price_4_nights,#numerical variables only selected, categorical not selected for simplicity
             host_listings_count,
             accommodates,
             bathrooms_number,
             bedrooms,
             beds,
             availability_30, #only keeping availablity in 30 days since the others add little value- tested using corr
             number_of_reviews,
             review_scores_rating,
             reviews_per_month,
             distance_opera) %>% 
  ggpairs(size=1)

listings_sydney_golden2

#in this part, we need to delete some high colineared variables(correlation >=0.7-according to the pearson correlation theory) beds and bedrooms has high correlation, meanwhile number of reviews and review per month have high correlation



```
1. We created a summary table, using `huxtable` that shows which models we worked on, which predictors are significant, the adjusted $R^2$, and the Residual Standard Error.

```{r hux table, cache=TRUE}

# produce summary table comparing models using huxtable::huxreg()
huxreg(model1, model2, model_4_variables,model_4_variables_final,model_superhost, model_instant_bookable,model_neighbourhood_cleansed,model_reviews,model_availability,model_reviews_availability,
       statistics = c('#observations' = 'nobs', 
                      'R squared' = 'r.squared', 
                      'Adj. R Squared' = 'adj.r.squared', 
                      'Residual SE' = 'sigma'), 
#       bold_signif = 0.05, 
       stars = NULL
) %>% 
  set_caption('Comparison of models')
```


```{r create our own final models, cache=TRUE, warning=FALSE}
#by adding more variables+ pervious ones like distance_opera,license, etc. and more categorical variables in final model with relatively highest adjusted R squared and ensure that all variables are siginificant


listings_sydney_golden_final<-listings_sydney_golden %>%
  mutate(host_response_rate=parse_number(host_response_rate),
         host_acceptance_rate=parse_number(host_acceptance_rate),
         amenities_number=length(list(amenities)))

for(i in 1:6111){
  listings_sydney_golden_final$amenities_words[i] <- lengths(strsplit(listings_sydney_golden_final$amenities[i],","))} #selecting the data in which we are interested

for(i in 1:6111){
  listings_sydney_golden_final$host_verification_words[i] <- lengths(strsplit(listings_sydney_golden_final$host_verifications[i],","))}#selecting the data in which we are interested

#best model currently we have
model_final<-lm(log_price_4_nights~number_of_reviews+bathrooms_number+bedrooms+review_scores_rating+room_type+host_response_rate+host_identity_verified+availability_90+review_scores_communication+review_scores_value+latitude+longitude+host_verification_words,data=listings_sydney_golden_final) 
msummary(model_final)
car::vif(model_final)
```


```{r create our own final models 2, cache=TRUE}
#only contains numeric variables
listings_sydney_golden_final_corr<-listings_sydney_golden_final%>%
  select(log_price_4_nights,
    number_of_reviews,
         bathrooms_number,
         bedrooms,
         review_scores_rating,
         host_response_rate,
         availability_90,
         review_scores_communication,
         review_scores_value,
         latitude,
         longitude,
         host_verification_words)

cor_listings_sydney<- cor(listings_sydney_golden_final_corr, method="pearson")
cor_listings_sydney

correl = cor(cor_listings_sydney, use="pairwise.complete.obs")
corrplot(cor_listings_sydney,method="color",type="lower",tl.cex=1)
corrplot(cor_listings_sydney,method="pie",type="upper",add=TRUE,tl.cex=1,cl.cex=0.5)


listings_sydney_golden2 <- listings_sydney_golden %>% 
  select(log_price_4_nights,#numerical variables only selected, categorical not selected for simplicity
             host_listings_count,
             accommodates,
             bathrooms_number,
             bedrooms,
             beds,
             availability_30, #only keeping availablity in 30 days since the others add little value- tested using corr
             number_of_reviews,
             review_scores_rating,
             reviews_per_month,
             distance_opera) %>% 
  ggpairs(size=1)

listings_sydney_golden2
```

```{r comparing all models}
#comparing majority of models created- removing a couple with lower R squared as only 9 are displayed
huxreg(model_superhost, model_instant_bookable,model_neighbourhood_cleansed,model_neighbourhood_cleansed2,model_reviews,model_availability,model_reviews_availability,model_final,
       statistics = c('#observations' = 'nobs', 
                      'R squared' = 'r.squared', 
                      'Adj. R Squared' = 'adj.r.squared', 
                      'Residual SE' = 'sigma'), 
#       bold_signif = 0.05, 
       stars = NULL
) %>% 
  set_caption('Comparison of all models')

```

1. Finally, we use the best model we came up with for prediction. Suppose you are planning to visit Sydney over reading week, and you want to stay in an Airbnb. We find Airbnb's in your destination city that are apartments with a private room, have at least 10 reviews, and an average rating of at least 90. We use our best model to predict the total cost to stay at this Airbnb for 4 nights. We can include the appropriate 95% interval with our prediction. 


Report the point prediction and interval in terms of `price_4_nights`. 
  - if you used a log_price_4_nights model, make sure you anti-log to convert the value in $. You can read more about [hot to interpret a regression model when some variables are log transformed here](https://stats.idre.ucla.edu/other/mult-pkg/faq/general/faqhow-do-i-interpret-a-regression-model-when-some-variables-are-log-transformed/)
#predict the total cost to stay at this Airbnb for 4 nights. Include the appropriate 95% interval with your prediction




```{r predict cost living in Sydney, cache=TRUE}
#Assume that an average rating of at least 90 means host_response_rate is greater than 90
imaginary_sydney_visit <-listings_sydney_golden_final%>%
  select(price_4_nights,
         log_price_4_nights,
         number_of_reviews,
         bathrooms_number,
         bedrooms,
         review_scores_rating,
         room_type,
         host_response_rate,
         host_identity_verified,
         availability_90,
         review_scores_communication,
         review_scores_value,
         latitude,
         longitude,
         host_verification_words)%>%
  drop_na()%>%
  filter(number_of_reviews>=10,room_type=="Private room",host_response_rate>=90)

predict_price<-exp(predict(model_final,newdata=imaginary_sydney_visit,interval="prediction",level=0.95))
predict_price

# there is no confidence interval here

```



# Deliverables

- By midnight on Monday 17 Oct 2022, you must upload on Canvas a short presentation (max 4-5 slides) with your findings, as some groups will be asked to present in class. You should present your Exploratory Data Analysis, as well as your best model. In addition, you must upload on Canvas your final report, written  using R Markdown to introduce, frame, and describe your story and findings. You should include the following in the memo:

1. Executive Summary: Based on your best model, indicate the factors that influence `price_4_nights`.
This should be written for an intelligent but non-technical audience. All
other sections can include technical writing.

When looking at our best model, the following factors stick out as those that significantly influence `price_4_nights`. The following had coefficients of magnitude greater than 0.1, i.e. if the following variables change then the our price would change a fair bit too: 

bathrooms_number
bedrooms
review_scores_rating
All categories in room_type
host_identity_verified
review_scores_communication
review_scores_value
latitude
longitude

Now from the above we can gather fairly expected qualitative factors for the airbnb market in Sydney- location, number of beds, number of baths, room type are all expected to influence the cost of the room. Similarly characteristics such as what rating the room has should too, as it is likely the more expensive rooms that are nicer. Whether a host identity is verified or not may influence price since those hosts that care enough to get verified are likely those that care more about using their property as a business venture and thus will keep nicer properties that charge more. 

2. Data Exploration and Feature Selection: Present key elements of the data, including tables and
graphs that help the reader understand the important variables in the dataset. Describe how the
data was cleaned and prepared, including feature selection, transformations, interactions, and
other approaches you considered.

From the initial dataset, we have tried to eliminate all repetitive or less significant variables. We have initially statistically analyzed the dataset to comprehend the situation at the beginning by utilizing skrim. Later on, we modified the dataset and started analyzing potential correlations resulting from the graphs. Specifically, it appears that correlations do not result in being linear. In order to do so, after having looked at the dataset through skim and glimpse, we also analyzed specific variables through favstats in order to get peculiar insights and understand the soundness of selected variables of the dataset. After that, we plotted through ggplot graphs to visualize some data. We modified the dataset by dropping not significant variables and identifying numerical variables to perform an analysis to identify their correlations. We did it by using corrplot and ggpairs.
The correlations demonstrate that the variables are not linearly related. Outside of the same typology of class (the different types of reviews, for instance), we have correlation coefficients that are quite low. This is because many variables are not linearly correlated even though.

Graphs can be found in the EDA section.

3. Model Selection and Validation: Describe the model fitting and validation process used. State
the model you selected and why they are preferable to other choices.

Progressively more variables were added to the models prior to reaching model_final. This model stuck out to us due to the large R-squared and significance of all numerical variables. We used the msummary function to validate this model, and compared this with our previously created models to also check that it was indeed the best. the neibhourhood_simplified factor was removed from our model due to it have a greater variability than what we accept (vif >5)

4. Findings and Recommendations: Interpret the results of the selected model and discuss
additional steps that might improve the analysis

In question 1, we discussed which variables were most resultant in changes in price. We were able to create a model which justified over two thirds of the changes in price. When fitting our model, our aim was to try and find every factor that had an affect on price and maximise our coverage of justifying the changes in price. In our final model, we ended up with fewer datapoints than we started with due to missing information in some of our factors. Although we still kept a very large portion of the data and kept the model statistically significant with the number of datapoints we had, we could have improved our analysis by keeping more of the data, even if it reduced our coverage of price change slightly.
  
# Assessment Rubric


```{r rubric, echo=FALSE, out.width="100%"}
knitr::include_graphics(here::here("images", "rubric.png"), error = FALSE)
```


# Acknowledgements

- The data for this project is from [insideairbnb.com](insideairbnb.com)