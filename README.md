# Zoom City Carbon Model::traffic CO2 emissions at street level using Machine Learning

<img width="1139" alt="git_zccm" src="https://user-images.githubusercontent.com/94705218/225165456-b2ad6b1e-9e59-4693-9e00-4371de475372.png">

Happy coding!

This document presents the **Zoom City Carbon Model**, an R-based tool designed to estimate CO2 emissions from road traffic at high spatio-temporal resolution. This model uses local traffic data, meteorological data, spatial data, and Machine Learning techniques (ML) to provide hourly estimates of traffic flow, average speed, and CO2 emissions at the road segment and whole street level. The code is divided into two parts: **Learn ML model** and **Deploy ML model**. The **LearnMLmodel** file is designed to train and test the ML-model, allowing users to assess the performance of the model for traffic estimates. The **DeployMLmodel** file deploys the ML to generate timeseries (.csv) and maps (.multipolylines) of CO2 emissions for your city.

To cite this model, please use: Anjos, M.; Meier, F. Zooming into City and tracking CO2 traffic emissions at street level.

## Input and setting requirements

To ensure the model runs correctly, the following inputs should be loaded:

1.  Traffic data .csv (required) with at least two columns called *date* and *id*.
2.  Counting traffic stations .csv or .shp (required) with at least three columns called *id*, *Latitude*, and *Longitude*.
3.  Meteorological data .csv (conditionally required) with at least one column called *date*.
4.  Other variables (optional), in .csv or .shp format, with the same date column recommendation.

The model converts the date-time into a R-formatted version, e.g., "2023-03-13 11:00:00" or "2023-03-13".

The following R packages should be installed in you PC:

```{r}

if (!require("pacman")) install.packages("pacman") # if the pacman package is not installed, install it
pacman::p_load(lubridate, tidyverse, data.table, sf, httr, openair, osmdata, tmap, recipes, timetk, caret, ranger, rmarkdown) # use pacman to load the following packages
library(lubridate) # A package that makes it easier to work with dates and times in R.
library(tidyverse) #A collection of packages for data manipulation and visualization, including dplyr, ggplot2, and tidyr
library(data.table) #A package for fast and efficient data manipulation.
library(sf) #A package for working with spatial data using the Simple Features (SF) standard.
library(httr)
library(openair) #A package for air quality data analysis and visualization.
library(osmdata) #A package for accessing and working with OpenStreetMap data.
library(tmap) #A package for creating static and interactive maps in R.
library(recipes) #A package for preprocessing data using a formula-based interface.
library(timeDate) #A package for working with dates and times in R.
library(timetk) #A package for manipulating time series data in R.
library(ranger) #A package for building fast and accurate random forests models.
library(caret) #A package for training and evaluating machine learning models in R.

```
Create a folder on your PC and define the path. Then, import the **ZCCM_functions.R** file which contains all the necessary functions.

```{r}
setwd("myFolder") #sets the working directory to the specified path
source("myFolder/ZCCM_functions.R") #runs the ZCCM_functions file, which contains all specific functions 

```

## Learn ML model

In this code, we create a ML model to estimate the hourly traffic flow and average speed at street level in Berlin, Germany. The data are the following:

-   Hourly volume of vehicles and average speed for different types of vehicles from the lane-specific detectors at 583 counting station from August to September 2022 from the Digital Platform City Traffic Berlin / Traffic Detection Berlin <https://api.viz.berlin.de/daten/verkehrsdetektion>. These data are named as *traffic_berlin_2022_08_09.csv* and *counting_stations_berlin.csv.*

-   Hourly meteorological data such as air temperature, relative humidity, sunshine, rainfall, wind direction, and wind speed from weather station Berlin-Dahlem (latitude 52.4537 and longitude 13.3017) managed by the German Weather Service Climate Data Center. The file is named as *weather_berlin_2022_08_09.csv*.

-   ESRI Shapefile describing the different land use classes in Berlin from the Berlin Digital Environmental Atlas <https://stadtentwicklung.berlin.de/umwelt/umweltatlas/edua_index.shtml>. The file is named as *var1_berlin_landuse.shp.*

The relevant files are stored as *traffic*, *stations*, *weather*, *var1*, *var2* and so on. In the traffic object, the columns for volume of vehicles and average speed should be renamed as *icars* and *ispeed*, respectively.

### Load data

```{r}
traffic <- fread("Data/traffic_berlin_2022_08_09.csv") %>% 
  dplyr::select(date, id, icars, ispeed)

stations_csv <- fread("Data/counting_stations_berlin.csv", dec=",") #Read cvs counting stations. 
stations <- sf::st_as_sf(stations_csv, coords = c("Longitude", "Latitude"), crs=4326) #Convert stations csv file to shapefile based on column Latitude and Longitude.

weather <- fread("Data/weather_berlin_2022_08_09.csv") %>%  #Read weather csv file
  dplyr::select(-V1) #Delete column

var1 <- sf::read_sf("shps/var1_berlin_landuse.shp")

```

### Get GIS features

Next, you need to obtain the road network for your city using the **getOSMfeatures** function. This function uses the osmdata package to download OpenStreetMap OSM features <https://wiki.openstreetmap.org/wiki/Map_features> and the sf package to convert them into spatial objects. It then geographically joins the OSM features (*iNetRoad*) and *var1* with road classes segments using the st_join and st_nearest_feature functions (*GIS_road*). It is recommend for users salving *iNetRoad* and *GIS_road* files.

```{r}
icity <- "Berlin"

my_area <- osmdata::getbb(icity, format_out = "sf_polygon", limit = 1)$multipolygon# Try this first option and plot to see the city 
my_area <- st_make_valid(my_area)
qtm(my_area)

# my_area <- osmdata::getbb(icity, format_out = "sf_polygon", limit = 1) #otherwise, try this one
# my_area <- st_make_valid(my_area)
# qtm(my_area)

class_roads <- c("motorway","trunk","primary", "secondary", "tertiary") #Define the road classes

iNetRoad <- getOSMfeatures(city = icity, 
                           road_class = class_roads, 
                           city_area = my_area, 
                           ishp = TRUE, #If TRUE, all feature shps are salved in the output folder. 
                           iplot = TRUE) #If TRUE, all feature maps are salved in the output folder. 
# st_write(iNetRoad, "myFolder/name.shp")
# iNetRoad <- st_read("myFolder/name.shp")

GIS_road <- st_join(iNetRoad, var1, join =st_nearest_feature, left = FALSE) #Join with var1, var2, var3 .....
#GIS_road <- st_join(GIS_road, var2, st_nearest_feature, st_is_within_distance, dist = 0.1)
#GIS_road <- st_join(GIS_road, var3, st_nearest_feature, st_is_within_distance, dist = 0.1)

```

### Roads categories

The next step is to divide all road segments into two categories: those with traffic count points, which we have labeled as "sampled", and those without, which we have labeled as "non-sampled". This task utilizes the previously obtained *iNetroad* and *GIS_road* object.

```{r}
road_sampled <- st_join(stations, GIS_road, join = st_is_within_distance, dist = 20, left = FALSE) %>%
  mutate(category = "sampled") %>% st_as_sf() %>% st_transform(crs = 4326)
road_nonsampled <- iNetRoad[!iNetRoad$osm_id%in%road_sampled$osm_id,]
road_nonsampled <- mutate(road_nonsampled, category = "nonsampled")
```

### Data splitting

The next step consists of dividing our dataset into two distinct sets: training and testing. First, we randomly assigned 80 % of our traffic count stations to the training set, 20 % to the test set. We made sure to distribute the number of stations evenly across different sampled road categories to ensure a representative sample (*fclass* defined in *class_road*). Next, we selected four months (August and September) from 2022, and split each month into the same training and testing sets. In the last task, we joined the split traffic with split counting stations by the column **id** to create *train_dataset* and *test_dataset*.

```{r}
stations_split <- road_sampled %>% distinct(id, .keep_all = TRUE) #create a dataframe with the unique station id
stations_split$fclass <- as.factor(stations_split$fclass) #change the factor class to a factor

set.seed(1232)
Index <- createDataPartition(stations_split$fclass, #create a data partition of the stations
                             p = 0.8, #80/20%
                             list = FALSE)
train_stations <- stations_split[ Index, ] #create a train and test dataframe
test_stations  <- stations_split[-Index, ]
df_split <- traffic %>% openair::selectByDate(year = 2022, month = 8:9) #Split up traffic timeseries 
df_split$split <- rep(x = c("training", "test"),
                      times = c(floor(x = 0.8 * nrow(x = df_split)), #80 % for training
                                ceiling(x = 0.2 * nrow(x = df_split)))) # 20 % for test
traffic_train <- df_split[df_split$split == 'training',] #create a train and test dataframe
traffic_test <- df_split[df_split$split == 'test',]

train_stations$id <- as.character(train_stations$id)
test_stations$id <- as.character(test_stations$id)
traffic_train$id <- as.character(traffic_train$id)
traffic_test$id <- as.character(traffic_test$id)

train_dataset <- inner_join(traffic_train, train_stations, by ="id") #create a traffic and stations by the "id".
test_dataset <- inner_join(traffic_test, test_stations, by ="id")

```

### Feature engineering and selection

This task resulted in *train_recipe* and *test_recipe* which contain all spatial and temporal features and dependent variables of the model. In the present example, the dependent variables are the mean of traffic flow (*icars*) and the mean speed (*ispeed*) at the road link (*oms_id*).

The *weather* object was joined by the column "*date*" (or other variables with date column).

It also involved imputing the missing values and transforming the data in order to select the most relevant predictors. Temporal predictors such as time of day, weekdays, weekends, and holidays indicators were generated using the Step_timeseries_signature function of the R package timetk, which converts the date-time column (e.g., 2023-01-01 01:00:00) into a set of indexes or new predictors.

```{r}

features_train <- train_dataset %>% #create a new dataframe with the train dataset
  group_by(date, osm_id) %>% #group by date and osm_id
  summarise(mean_cars = round(mean(icars),digits = 0), #calculate the mean of the cars  and mean speed as depend variables
            mean_speed = round(mean(ispeed), digits = 0), .groups = "drop") %>% 
  filter(mean_cars>10, mean_speed > 10) %>% #filter the dataframe by the mean of cars and speed
  inner_join(road_sampled, by= "osm_id") %>% #join the road sampled dataframe
  inner_join(weather, by= "date") %>% #join the weather dataframe
  as_tibble() %>% dplyr::select(-Latitude, -Longitude, -id, -name, -osm_id,-category, -geometry) %>% #Drop the unsual features
  mutate_if(is.character, as.factor) #mutate the character variables to factor

features_test <- test_dataset %>% #create a new dataframe with the test dataset
  group_by(date, osm_id) %>% #group by date and osm_id
  summarise(mean_cars = round(mean(icars),digits = 0), #calculate the mean of the cars and round to 0 digits
            mean_speed = round(mean(ispeed), digits = 0), .groups = "drop") %>% #calculate the mean of the speed and round to 0 digits
  filter(mean_cars>10, mean_speed > 10) %>% #filter the dataframe by the mean of cars and speed
  inner_join(road_sampled, by= "osm_id") %>% #join the road sampled dataframe
  inner_join(weather, by= "date") %>% #join the weather dataframe
  as_tibble() %>% dplyr::select(-Latitude, -Longitude, -id, -name, -osm_id, -category, -geometry) %>% #select the features
  mutate_if(is.character, as.factor) #mutate the character variables to factor

receipe_steps <-
  recipe(mean_cars + mean_speed ~., data = features_train) %>% # Depend variable selected
  step_ts_impute(all_numeric()) %>% #Impute values for numeric predictors and outcomes
  step_impute_mode(lanes, maxspeed) %>% #Impute values for nominal/categorical variables_cars
  step_unknown(all_nominal_predictors()) %>%
  step_other(all_nominal_predictors(), -lanes, -maxspeed) %>%
  step_timeseries_signature(date) %>% # creating indexes from date-time
  step_rm(date, contains("index.num"), contains("iso"), contains("xts")) 

train_recipe <- receipe_steps %>% # create a recipe for the training data
  prep(features_train) %>%
  bake(features_train)

test_recipe <- receipe_steps %>% # create a recipe for the test data
  prep(features_test) %>%
  bake(features_test)

```

### Selection and training of ML algorithm

To train and test the Ml model, we used Random Forest (RF), a popular ensemble learning technique known for its ability to combine a large number of decision trees for classification or regression.The ranger R package was used to run the RF for traffic flow and speed predictions.

```{r}

train_processed <- train_recipe %>% #Training the RF for traffic flow predictions
  dplyr::select(-mean_speed) #Delete mean_speed for training and test sets as RF runs the traffic flow
test_processed <- test_recipe %>%
  dplyr::select(-mean_speed)

set.seed(1234)
rfModel_cars <- ranger(dependent.variable.name = "mean_cars",
                       data = train_processed, 
                       num.trees = 100, 
                       importance = "permutation") 

rfModel_pred_cars <- predict(rfModel_cars, data=test_processed) #Make predictions for test data with trained model

rfModel_df_cars <- rfModel_pred_cars$predictions %>% #Create the new dataframe with predictions and other variables
  bind_cols(features_test %>% dplyr::select(date)) %>%
  bind_cols(test_processed) %>%
  rename(predcars = ...1)
write_csv(rfModel_df_cars, "rfModel_df_cars.csv")

train_processed <- train_recipe %>% #Training the RF for average speed predictions
  dplyr::select(-mean_cars) #Delete mean_cars for training and test sets as RF runs the average speed 
test_processed <- test_recipe %>%
  dplyr::select(-mean_cars)

set.seed(1234)
rfModel_speed <- ranger(dependent.variable.name = "mean_speed",
                        data = train_processed, 
                        num.trees = 100,
                        importance = "permutation") 

rfModel_pred_speed <- predict(rfModel_speed, data=test_processed)

rfModel_df_speed <- rfModel_pred_speed$predictions %>% 
  bind_cols(features_test %>% select(date)) %>%
  bind_cols(test_processed) %>%
  rename(predspeed = ...1)
write_csv(rfModel_df_speed, "rfModel_df_speed.csv")

```

### Model Evaluation and interpretabilty

As the RF is part of black-box models, we utilized the feature importance method (*permutation*), which measures the contribution of each feature to the final predictions. The openair R package <https://bookdown.org/david_carslaw/openair/> was used to generate the plots and calculate the metrics from the *rfModel_df_cars* and *rfModel_df_speed*.

#### ML model for traffic flow predictions

```{r}

rfModel_df_cars %>% #Plot timeseries for observed and modelled values
  openair::timePlot(pollutant = c("mean_cars", "predcars"), group = TRUE,
           avg.time = "hour",  name.pol = c("Observed", "ML-model"),
           auto.text = TRUE, cols = c("#4a8bad", "#ffa500"),
           fontsize = 16, lwd = 2, lty = 1,
           ylab = "Traffic flow",
           main = "")

rfModel_df_cars %>% #Plot time variation 
  openair::timeVariation(pollutant = c("mean_cars", "predcars"),
                name.pol = c("Observed", "Modelled"),
                cols = c("#FAAB18", "#1380A1"),
                ci = TRUE, lwd = 3,
                fontsize = 14, ylim = c(0, 800), key.position = "bottom",
                ylab ="Traffic flow")

metrics_cars <- openair::modStats(rfModel_df_cars, mod= "predcars", obs = "mean_cars", type = "hour") #It provides a set of metrics by hour by the argument "type". 
write_csv(metrics_cars, "metrics_cars.csv") #Salve the metrics table.

#variable importance
variables_cars <- as.data.frame(importance(rfModel_cars), type = 1)
colnames(variables_cars) [1] <- "importance"
variables_cars<- cbind(var.names = rownames(variables_cars), variables_cars)
variables_cars<- mutate(variables_cars, importance = importance / sum(importance) * 100,
                   importance = round(importance, digits = 1)) %>%
  arrange(desc(importance))
write_csv(variables_cars, "importance_cars.csv")

variables_cars %>%
  head(20) %>% #Top 20 features
  ggplot(aes(x=reorder(var.names, importance), y=importance, fill=importance))+
  geom_bar(stat="identity", position="dodge", show.legend = FALSE)+
  ylab("Contribution (%)")+
  coord_flip() +
  xlab("Top 20 features")+
  labs(
    subtitle = "traffic flow predicitions") +
  geom_text(aes(label = importance), hjust = 0, size = 5) +
  #scale_y_continuous(limits = c(0, 20)) +
  scale_fill_viridis_c(direction = -1) +
  theme_classic(base_size = 15)
#ggsave("importance_cars_plot.png", iplot) #Salve the plot

```

#### ML model for average speed predictions

```{r}

rfModel_df_speed %>% #plot the timeseries
  openair::timePlot(pollutant = c("mean_speed", "predspeed"), group = TRUE,
           avg.time = "hour", name.pol = c("Observed", "ML-model"),
           auto.text = TRUE, cols = c("forestgreen", "brown2"),
           fontsize = 16, lwd = 2, lty = 1,
           ylab = "Average speed [km/h]",
           main = "")
metrics_cars <- modStats(rfModel_plot_cars, mod= "predspeed", obs = "mean_speed", type = "hour") #get the metrics and assess your model
write_csv(metrics_cars, "metrics_speed.csv")

variables_speed <- as.data.frame(importance(rfModel_speed), type = 1)
colnames(variables_speed) [1] <- "importance"
variables_speed<- cbind(var.names = rownames(variables_speed), variables_speed)
variables_speed<- mutate(variables_speed, importance = importance / sum(importance) * 100,
                        importance = round(importance, digits = 1)) %>%
  arrange(desc(importance))
write_csv(variables_speed, "importance_speed.csv")

variables_speed %>%
  head(20) %>% #Top 20 features
  ggplot(aes(x=reorder(var.names, importance), y=importance, fill=importance))+
  geom_bar(stat="identity", position="dodge", show.legend = FALSE)+
  ylab("Contribution (%)")+
  coord_flip() +
  xlab("Top 20 features")+
  labs(
    subtitle = "Average speed predicitions") +
  geom_text(aes(label = importance), hjust = 0, size = 5) +
  #scale_y_continuous(limits = c(0, 20)) +
  scale_fill_viridis_c(direction = -1, option = "E") +
  theme_classic(base_size = 15)

```

# Deploy ML model

After fine-tuning and evaluating the ML model on the dataset, it is deployed to predict traffic flow and average speed for each road segment in the city using the  **DeployMLtraffic** function. This function calculates traffic CO2 emissions at the street level and produces time series and maps of traffic predictions and CO2 emissions.

To use the **DeployMLtraffic** function, you need to input data such as *traffic,* *stations*, and *weather*, as well the *GIS_road* object obtained from the Get OSM features section. The function performs all the steps described in the Lean ML model, except data splitting and model evaluation.

The **DeployMLtraffic** has several arguments, including:

-   **input** a data frame with the period defined as months and years for calculations.

-   **traffic_data**, **stations_data**, and **weather_data** :inputs for the function.

-   **road_data** a shapefile that describes the road segments with OSM features, which is named *GIS_road* in this case.

-   **n.trees** number of decision trees in Rondam Forest. The default is 100. 

-   **cityStreet**: if TRUE, the function calculates all prediction values on each road segment within the city area and provides a dataframe.Rds for each day in the output_cityStreet folder.

-   **cityCount**: if TRUE, the function sums up all prediction values within the city area and provides a dataframe.csv for each day in the output_citycount folder.

-   **cityMap**: if TRUE, the function calculates all prediction values on each road segment within the city area and provides a stack raster with 100 meters resolution in a .tiff and shapfile .GPKG formats for each day in the output_citymap folder.

-   **tempRes**: the temporal resolution, which can be "sec", "min", "hour", "day", "DSTday", "week", "month", "quarter", or "year".

-   **spatRes**:: the spatial resolution of the cityMap. The default is 100 meters.

-   **inuit**: the unit of cityMap for CO2 emissions, which can be "micro" (CO2 emissions [micro mole per meter, square per second]), "grams" (CO2 emissions [grams per meter]), or "gramsCarbon" (Carbon emissions [grams per meter]). Note that cityStreet and cityCount includes all units.

-   **ista**: the statistic to apply when aggregating the data, which can be "mean", "max", "min", "median", "frequency", "sd", or "percentile". The default is the sum.

Once all arguments are defined, the **DeployMLtraffic**  function can be run using the apply function for the selected period. In this example, the result is stored as *myMLtraffic*. If cityCount is TRUE and cityMap is FALSE, the do.call function can be used to merge the list of days of *myMLtraffic* into a unique dataframe with the complete time series, which is named *CO2_count*. If cityCount is FALSE and cityMap is TRUE, the unlist function can be used to obtain the stack raster, which is named *CO2_map*.

```{r}

#define the period (inputDates)
#imonth <- c("jan", "feb", "mar", "apr", "may", "jun", "jul", "aug", "sep", "oct", "nov", "dec") or c(1:12)
#iyear <- c(2015:2020), c(2015, 2017, 2020) or (2020).

imonth <- c("aug","sep")
iyear <- c(2022)
input <- expand.grid(imonth, iyear) 

#Applying the DeployML traffic function for the selected period
myMLtraffic <- pbapply::pbapply(input, 1, 
                                DeployMLtraffic(city="Berlin", input, 
                                                traffic_data = traffic, 
                                                stations_data = stations,
                                                weather_data = weather, 
                                                road_data = iNetRoad,
                                                n.trees = 100,
                                                cityStreet = TRUE, 
                                                cityCount = TRUE, 
                                                cityMap = TRUE, 
                                                tempRes = "hour", 
                                                spatRes = 100, 
                                                iunit = "grams", 
                                                ista = "sum")) 

#Take your timeseries and salve based on the selected arguments:
CO2_street <- do.call(rbind.data.frame, myMLtraffic) #Use for the cityStreet 
saveRDS(CO2_street, "CO2_street_Berlin_2022_08_09.rds") #salve file

CO2_count <- do.call(rbind.data.frame, myMLtraffic) #Use for the cityCount 
write_csv(CO2_count, "CO2_count_Berlin_2022_08_09.csv") #salve file

CO2_map <- unlist(myMLtraffic) ##Use for the cityMqp
raster::writeRaster(CO2_map,"CO2_map_Berlin_2022_08_09.TIF", format="GTiff", overwrite =TRUE) #salve file

```

