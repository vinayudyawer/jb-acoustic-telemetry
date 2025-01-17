---
title: "Summarising and Plotting Data"
teaching: 45
exercises: 0
questions:
- "What are some of the ways to summarise my data?"
- "How can I make plots of the summarised data?"
objectives:
- "Show how to summarise data by FishID."
- "Show how to summarise data by station."
- "Use ggplot and ggmap to display plots and maps of the data."
- "Use ggmap to track individual fish movement."
keypoints:
- "Data can be summarised by fish ID, station or detection."
- "Data can be plotted into an abacus plot by day, or can be mapped based on the station information."
---

## Summarise by FishID

We can group our data by animal ID, letting us isolate individuals for summary stats, plotting and further analysis.
~~~
str(dets_with_stations)
animal_id_summary <- dets_with_stations %>% group_by(animal_id) %>%
  summarise(dets=length(animal_id),stations=length(unique(station)),
            min=min(detection_timestamp_utc), max=max(detection_timestamp_utc), 
            tracklength=max(detection_timestamp_utc)-min(detection_timestamp_utc)) %>% as.data.frame()
animal_id_summary

~~~
{:.language-r}

## Summarise by station:

To group and summarise our data by station, we first need to find all unique station names and locations:
~~~
head(Rxdeploy)
stations <- Rxdeploy %>% select(station, deploy_date_time, recover_date_time, lat=deploy_lat, lon=deploy_long)
head(stations)
~~~
{:.language-r}

## Summarise detections:
Create a new data product, det_days, that give you the unique dates that an animal was seen by a station. This summary
can be used to calculate a residence index as in [Kessel et al. 2017](https://dx.doi.org/10.1007/s00300-015-1723-y)
~~~
stationsum <- dets_with_stations %>% group_by(station) %>%
  summarise(detections=length(animal_id),
            start=min(detection_timestamp_utc),
            end=max(detection_timestamp_utc),
            uniqueID=length(unique(animal_id)), det_days=length(unique(as.Date(detection_timestamp_utc)))) %>% as.data.frame()
~~~
{:.language-r}

## Merge with station list:

We can then re-attach station summary information to the list of stations we made earlier.

~~~
stations2 <- merge(stations, stationsum, all.x=TRUE, by="station")
stations2 <- stations2 %>% filter(detections > 0) # Filter out stations with no detections
stations2 <- stations2 %>% filter (deploy_date_time <= start & recover_date_time >= end) %>% select(-start, -end)
stations2[1:10,]
~~~
{:.language-r}


## Plotting data

Abacus plots:

Simple plots using the glatos library.

~~~
library(glatos)

glatos::abacus_plot(dets_with_stations, location_col =  "station") # Plot by station
glatos::abacus_plot(dets_with_stations, location_col =  "animal_id") # Plot by animal
~~~
{:.language-r}

Nicer plots using the ggplot library.

~~~
library(dplyr)
library(ggplot2)
library(viridis)

plot_data <- dets %>% dplyr::select(animal_id, station, detection_timestamp_utc)

abacus_animals <- ggplot(data=plot_data, aes(x=detection_timestamp_utc, y=animal_id, col=station))+
  geom_point()+
  ggtitle("Detections by animal")+
  theme(plot.title = element_text(face="bold", hjust = 0.5))+
  scale_color_viridis(discrete= TRUE)

abacus_animals

abacus_stations <- ggplot(data=plot_data,  aes(x=detection_timestamp_utc, y=station, col=animal_id))+
  geom_point()+
  ggtitle("Detections by station")+
  theme(plot.title = element_text(face="bold", hjust = 0.5))+
  scale_color_viridis(discrete= TRUE)

abacus_stations
~~~
{:.language-r}


## Spatial Plots:

Summarise data by station and individual ID, and then plot a map of each animal path.

~~~
library(ggmap)

# examine by station and FishID:
stationFishID <- dets_with_stations %>% group_by(station, animal_id) %>%
  summarise(lat=mean(deploy_lat), lon=mean(deploy_long), dets=length(animal_id), logdets=log(length(animal_id)))

# Peek at the first few rows
head(stationFishID)

base <- get_stamenmap(
  bbox = c(
    left = min(dets_with_stations$deploy_long),
    bottom = min(dets_with_stations$deploy_lat ), 
    right = max(dets_with_stations$deploy_long), 
    top = max(dets_with_stations$deploy_lat)
  ),
  maptype = "terrain-background", 
  crop = FALSE,
  zoom = 8
)
perm_map <- ggmap(base, extent='normal')+
  ylab("Latitude") +
  xlab("Longitude")+
  labs(size="log(detections)")+
  geom_point(data=stationFishID, aes(x=lon,y=lat,size=logdets,col=animal_id))
perm_map

# can simply save plots using output window, or to save high res plots:
tiff("Keys_permit_map.tiff", units="in", width=15, height=8, res=300)
perm_map
dev.off()
~~~
{:.language-r}


## Using maps to track movement patterns

Let's look at maps by FishID to see individual fish movement patterns.

~~~
lat_range <- range(dets_with_stations$deploy_lat) + c(-1, 1)
lon_range <- range(dets_with_stations$deploy_long) + c(-1, 1)

GLmap <- get_stamenmap(
  bbox = c(left = lon_range[1],
    bottom = lat_range[1],
    right = lon_range[2],
    top = lat_range[2]),
  maptype = "terrain", 
  crop = FALSE,
  zoom = 6)

movMap <- ggmap(GLmap, extent='normal')+
  coord_cartesian(xlim=lon_range, ylim=lat_range)+
  ylab("Latitude") +
  xlab("Longitude")+
  labs(size="log(detections)")+
  geom_path(data=dets_with_stations, aes(x=deploy_long, y=deploy_lat, col=animal_id))+
  geom_point(data=stationFishID, aes(x=lon,y=lat,size=logdets,col=animal_id))+
  facet_wrap(~animal_id)
movMap
~~~
{:.language-r}
