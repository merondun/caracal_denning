# Data Import and Prep

First, import GPS location data and denning data, and assign each GPS location a cluster ID based on date time: 

Use these den data:

| ID    | Cluster | Latitude | Longitude | Den_Start       | Den_End          | Dist_Urban |
| ----- | ------- | -------- | --------- | --------------- | ---------------- | ---------- |
| TMC27 | 1       | -34.0592 | 18.33856  | 9/11/2016 20:00 | 10/25/2016 11:00 | 0.32       |
| TMC28 | 1       | -34.3503 | 18.47401  | 9/10/2016 0:00  | 10/6/2016 6:00   | 14.46      |
| TMC28 | 2       | -34.3487 | 18.47383  | 10/5/2016 0:00  | 10/13/2016 15:00 | 14.16      |
| TMC28 | 3       | -34.3456 | 18.46527  | 10/14/2016 9:00 | 10/25/2016 18:00 | 13.6       |
| TMC13 | 1       | -34.1182 | 18.41024  | 9/24/2015 18:00 | 10/9/2015 9:00   | 0.6        |
| TMC13 | 2       | -34.1171 | 18.41102  | 10/10/2015 0:00 | 11/5/2015 21:00  | 0.6        |
| TMC03 | 1       | -34.1048 | 18.35837  | 1/1/2016 8:00   | 1/29/2016 8:00   | 0.41       |

```R
# This script takes in the individual .csv data and calculates centroids for each based on
# Harversine distance between lat/long pairs. Potential sensitivity in parameters:
# eps = reachability, in km; and minpts = reachability min. points, see Ester et al 1996
# for more details and ?dbscan for details on the function

# Set basic stuff we need on the cluster rstudio
setwd('C:/Users/herit/My Drive/Research/Caracal/Denning/2023JULY')
#setwd('~/merondun/misc_research/caracals/')
#.libPaths('~/mambaforge/envs/caracals/lib/R/library')
set.seed(111)

library(tidyverse)
library(geosphere)
library(fpc)
library(viridis)
library(sf)
library(lubridate)
library(ggdist)
library(ggpubr)
library(gghalves)

#read in caracal data
files = list.files('.',pattern='csv',full.names = TRUE)
df = NULL
for (file in files) {
  d =  read.csv(file,header=TRUE) %>% as_tibble
  df = rbind(df,d)
}

##### Data Prep #####

# Transform Date_time and calculate time differences
df = df %>%
  mutate(Date_time = paste0(Date,' ',Time),
         Date_time = mdy_hms(Date_time)) %>%
  arrange(ID, Date_time) %>%
  group_by(ID) %>%
  mutate(Time_diff = c(0, diff(Date_time))) %>% 
  select(-c(Date,Time))

# Import den data
dens = read_tsv('Den_Locations_Laurel.txt')
dens = dens %>% 
  mutate(Date_time_start = mdy_hm(Den_Start),
         Date_time_end = mdy_hm(Den_End)) %>% select(-c(Den_Start,Den_End)) %>% 
  dplyr::rename(Den_Lat = Latitude, Den_Long = Longitude)

# Merge the data frames based on ID
df_merged = left_join(df, dens, by = "ID")

# Assign clusters based on the date-time range
df_den = df_merged %>%
  mutate(Cluster = ifelse(between(Date_time, Date_time_start, Date_time_end), Cluster, NA))

# Summary stats on dens 
dfp = df_den %>% drop_na(Cluster) %>% 
  group_by(ID,Cluster) %>% 
  summarize(Total_time = difftime(max(Date_time), min(Date_time), units = "hours"),
            Observations = n()) %>%
  ggplot(aes(x=Total_time,y=Observations,col=as.factor(Cluster),shape=ID))+
  geom_point(size=4)+xlab('Total Time Spent at Den (Hours)')+ylab('Number of GPS Locations')+
  scale_color_viridis('Den',discrete=TRUE,option='turbo')+
  theme_bw()

# ID    Cluster Total_time Observations
# <chr>            <dbl> <drtn>            <int>
#   1 TMC03                1  672 hours          221
# 2 TMC13                1  348 hours          116
# 3 TMC13                2  642 hours          210
# 4 TMC27                1 1047 hours          239
# 5 TMC28                1  627 hours          183
# 6 TMC28                2  204 hours           55
# 7 TMC28                3  270 hours           88

png('Den_Summaries.png',height=5,width=6,units='in',res=400)
dfp
dev.off()

# Calculate Haversine distance for each point from the cluster centroid 
df_den_filt <- df_den %>%
  drop_na(Cluster) %>% 
  rowwise() %>%
  mutate(dist_to_den = distm(cbind(Longitude, Latitude),
                                  cbind(Den_Long, Den_Lat),
                                  fun = distHaversine)) # distance in meters

#Summaries of distance 
dendist = df_den_filt %>%
  ggplot(aes(x=interaction(ID, Cluster), y=dist_to_den,fill=as.factor(Cluster), col=as.factor(Cluster))) +
  ggdist::stat_halfeye(adjust = .5,width = .6,.width = 0,justification = -.2, point_colour = NA,alpha = 0.9,normalize='groups')+
  gghalves::geom_half_point(aes(col=as.factor(Cluster)),side='l',range_scale = .4,alpha = .3)+
  scale_fill_viridis('Den ID',discrete=TRUE, option='turbo') +
  xlab('ID + Den ID')+ ylab('Distance to Den (m)')+
  scale_color_viridis('Den ID',discrete=TRUE, option='turbo') +
  facet_grid(.~ID,scales='free')+
  theme_bw()

png('Den_Summaries.png',height=5,width=12,units='in',res=400)
ggarrange(dfp,dendist,common.legend = TRUE,widths = c(0.3,0.7))
dev.off()

write.table(df_den_filt,file='Caracal_Denning_Data_2023JULY28.txt',quote=F,sep='\t',row.names=F)

# And also save a KML with the points
df_sf = st_as_sf(dens, coords = c("Den_Long", "Den_Lat"), crs = 4326)
df_sf %>% st_write(paste0("Centroids.kml"),append=FALSE)
```

# Identify Trips

Then identify trips

```R
# This script takes the output of Cluster_Locations.R and identifies trips
# Potential parameter sensitivity in the distance away from centroid a track is considered
# a 'trip'

# Set basic stuff we need on the cluster rstudio
setwd('G:/My Drive/Research/Caracal/Denning/2023JULY/')
#setwd('~/merondun/misc_research/caracals/')
#.libPaths('~/mambaforge/envs/caracals/lib/R/library')
set.seed(111)
options(scipen = 999)

library(tidyverse)
library(geosphere)
library(viridis)
library(sf)

##### Calculate some metrics #####
# We will base all these analyses on a 'trip'
df_den_filt = read.table('Caracal_Denning_Data_2023JULY28.txt',header=TRUE,sep='\t')  %>%
  as_tibble %>%
  mutate(Date_time = ymd_hms(Date_time))

threshold = 125 #meters away from centroid until a trip starts

# Define the start and end of a trip. A trip starts when the individual is
# more than n meters from the cluster centroid, and ends when the individual
# returns within n meters of the centroid, not including the distances before / after 
# the individual was within the threshold
df_den_filt = df_den_filt %>% 
  group_by(ID,Cluster) %>% 
  mutate(away = ifelse(dist_to_den > threshold,1,0),
         trip = ifelse(away == 1, data.table::rleid(away), NA))

# The numbering of the rleid command will count 1..3..5 due to the succession of 0 and 1 aways, so add a unique ID for each trip  
trips = df_den_filt %>% select(ID,Cluster,trip) %>% drop_na(trip) %>% unique %>% group_by(ID,Cluster) %>% mutate(trip_ID = paste0(ID,'_Den',Cluster,'_Trip',row_number())) 

# Bind into a final frame, points not involved a trip will be 'NA'
df_trips = left_join(df_den_filt %>% select(-away),trips) %>% select(-c(trip)) 

# Calculating the distance between subsequent points, we will need this for total distance for each trip 
df_trips = df_trips %>%
  group_by(ID,Cluster) %>%
  arrange(Date_time) %>%
  mutate(
    lat_lag = lag(Latitude, default = first(Latitude)),
    lon_lag = lag(Longitude, default = first(Longitude)),
    dist = distHaversine(cbind(lon_lag, lat_lag), cbind(Longitude, Latitude))) %>%  #using geosphere calculate distance between points, used for total_distance 
  group_by(ID,Cluster,trip_ID) %>% 
  mutate(dist1 = if_else(row_number() == 1, dist_to_den, dist)) %>% # BUT, for the initial trip start, we DO want to use distance from centroid, since we need our initial distance 
  mutate(dist2 = if_else(row_number() == n(), dist + dist_to_den, dist1)) %>% #AND for the final trip end, we ADD the distance_to_centroid to the distance from the last segment 
  ungroup()

# Check that things are sensible with df_trips %>% print(n = 100), and then drop the columns we don't need
df_trips = df_trips %>% select(-c(dist,lat_lag,lon_lag,dist1)) %>% dplyr::rename(dist=dist2)
df_trips %>% print(n = 50)

# Summarize those trips! 
trip_summaries <- df_trips %>%
  group_by(ID, Cluster, trip_ID) %>%
  summarise(
    trip_start = min(Date_time),
    max_dist = max(dist_to_den, na.rm = TRUE),
    total_dist = sum(dist, na.rm = TRUE),
    duration = max(Date_time) - min(Date_time),
    observations = n(), #number of points
    min_hours = as.numeric(duration / 60 / 60) + 0, # minimum possible time spent in a cluster (2 observations could be 3 hours outside cluster)
    avg_hours = as.numeric(duration / 60 / 60) + 3, # average possible time spent in a cluster (2 observations could be 6 hours outside cluster)
    max_hours = as.numeric(duration / 60 / 60) + 6, # maximum possible time spent in a cluster (2 observations could also be 9 hours outside cluster)
  ) %>% ungroup %>% 
  group_by(ID,Cluster) %>% 
  mutate(proportion = (avg_hours/sum(avg_hours)))  # Proportion of time spent on each trip compared to the total

# How many trips for each caracal
trip_summaries %>% 
  drop_na(trip_ID) %>%
  ggplot(aes(x=ID,fill=as.factor(Cluster)))+
  geom_bar(width=1,position=position_dodge(width=1.1))+
  scale_fill_viridis(discrete=TRUE,option='turbo')+
  facet_grid(.~ID,scales='free')+
  ggtitle('Total Number of Trips')+
  xlab('')+ylab('Number of Trips')+
  theme_bw()

# Inspect a few manually, rebind with the original frame
trip_summaries %>% arrange(desc(avg_hours)) 
full_trip = left_join(df_trips %>% dplyr::rename(segment_distance = dist),trip_summaries %>% select(-c(trip_start,duration)))
catname = data.frame(ID = c('TMC03','TMC13','TMC27','TMC28'), Name = c('Fire Lily','Hope','Disa','Luna'))

ft = left_join(full_trip,catname)

write.table(ft,'Caracal_Denning_Data_Trips-125m-HourRange_2023AUG02.txt',quote=F,sep='\t',row.names=F)

```

# Analyze Trips

```R
#Analyze some of the trip information 

# Set basic stuff we need on the cluster rstudio
setwd('G:/My Drive/Research/Caracal/Denning/2023JULY')
#setwd('~/merondun/misc_research/caracals/')
#.libPaths('~/mambaforge/envs/caracals/lib/R/library')
set.seed(111)

library(tidyverse)
library(geosphere)
library(viridis)
library(sf)
library(ggmap)

full_trip = read.table('Caracal_Denning_Data_Trips-125m-HourRange_2023AUG02.txt',header=TRUE,comment.char='',sep='\t') %>% 
  as_tibble %>% 
  group_by(ID,trip_ID) %>% 
  mutate(Date_time = ymd_hms(Date_time), #ensure date in interpreted correctly
         trip_start = min(Date_time)) %>% #add the beginning of the trip 
  ungroup

##### Trip Analysis ##### 

# First explore raw trip data 
trip_data = full_trip %>% select(ID,Cluster,trip_start,trip_ID,max_dist,total_dist,avg_hours,min_hours,max_hours) %>% unique %>%
  drop_na(trip_ID)

trip_data %>% 
  pivot_longer(!c(ID,Cluster,trip_start,trip_ID)) %>% 
  ggplot(aes(x=trip_start,col=as.factor(Cluster),y=value))+
  geom_point()+
  #geom_bar(stat='identity',position=position_dodge())+
  scale_color_viridis(discrete=TRUE,option='turbo')+
  facet_grid(name~ID,scales='free')+
  ggtitle('Trip Duration & Distance')+
  xlab('Date')+
  theme_bw()

#Generates the table found on github, show the 2 longest trips for each caracal (HOURS)
trip_data %>% group_by(ID) %>% slice_max(avg_hours, n = 2) 

#Generates the table found on github, show the 2 longest trips for each caracal (MAX DISTANCE) 
trip_data %>% group_by(ID) %>% slice_max(max_dist, n = 2) 

#Generates the table found on github, show the 2 longest trips for each caracal (TOTAL DISTANCE) 
trip_data %>% group_by(ID) %>% slice_max(total_dist, n = 2) 

#Apply a minimum number of observations to call it a trip, if necessary 
trip_data = trip_data %>% filter(avg_hours >= 0 )

# Plot Some Summaries of Trips 
tidy_trip <- trip_data %>%
  mutate(max_dist = max_dist/1000,  #conver to km from m 
         total_dist = total_dist/1000) %>% 
  #mutate(across(c(avg_hours, max_dist, total_dist),  #this command will replace any observations above the 99% IQR with NA to deal with outliers
  #              ~if_else(.x > quantile(.x, 0.99), NA_real_, .x))) %>% 
  dplyr::rename('Max Distance (km)' = max_dist,'Total Distance (km)' = total_dist,'Total Hours (h)' = avg_hours) %>% 
  pivot_longer(!c(ID,Cluster,trip_ID,trip_start)) %>% drop_na(value)

# Plots 
tidy_trip %>% ggplot(aes(x=trip_start,col=as.factor(Cluster),y=value,shape=name))+
  geom_point()+
  #geom_bar(stat='identity',position=position_dodge())+
  scale_color_viridis(discrete=TRUE)+
  facet_grid(name~ID,scales='free')+
  ggtitle('Trip Duration & Distance')+
  xlab('')+ylab('')+
  theme_bw()+
  theme(axis.text.x=element_text(angle=45,vjust=1,hjust=1))

tidy_trip = tidy_trip %>% ungroup %>%  
  group_by(ID) %>% 
  mutate(days_since_start = as.numeric(difftime(trip_start, min(trip_start), units = "days")))

# Simply calculate correlations between date start and distance / hours 
stats <- tidy_trip %>% 
  group_by(ID, name) %>%  #undetermined if should add cluster grouping if we want to 're-initiate' denning 
  summarize(rho = cor.test(days_since_start, value, method='spearman')$estimate,
            p.value = cor.test(days_since_start, value, method='spearman')$p.value) %>% 
  mutate(p.value.adj = p.adjust(p.value, method = "bonferroni", n = nrow(results)))


# We will add a simple x-axis label to the beginning date for each ID
dates = tidy_trip %>% group_by(ID) %>% slice_max(trip_start,n=1) %>% ungroup %>% select(-c(value,trip_ID,name)) %>% unique
stats = left_join(stats,dates,relationship='many-to-many')
# Add label for if is signifcant (0.05)
stats = stats %>% mutate(signiflab = ifelse(p.value.adj < 0.05,'*','n.s.'),
                         label = paste0(rho))

# Plot the points and the labels 
trip_stats = tidy_trip %>% ggplot(aes(x=trip_start,col=as.factor(Cluster),y=value,shape=name))+
  geom_point()+
  geom_text(data=stats,aes(x=trip_start,y=Inf,label=paste0("r: ", signif(rho, 2),' ',signiflab)),
            size=2.5,vjust=1,col='black',hjust=1)+
  scale_color_viridis('Den ID',discrete=TRUE)+
  scale_shape_manual('Variable',values=c(16,17,15))+
  geom_smooth()+
  facet_grid(name~ID,scales='free')+
  ggtitle('Trip Duration & Distance')+
  xlab('')+ylab('')+
  theme_bw()+
  theme(axis.text.x=element_text(angle=45,vjust=1,hjust=1))
trip_stats

# pdf('Caracal_Trip_Stats_125m-2023JULY27.pdf',height=6,width=9)
# trip_stats
# dev.off()
# 
# write_tsv(trip_data,'Caracal_Trip_Stats_125m-2023JULY27.txt')

#linear models 
library(broom) 

results = tidy_trip %>%
  group_by(ID, name) %>%
  do(tidy(lm(value ~ days_since_start, data = .))) %>%
  filter(term == "days_since_start") %>%
  mutate(p.value.adj = p.adjust(p.value, method = "bonferroni", n = nrow(results)))
results

```

 The 2 LONGEST (HOURS) Trips by caracal:

| ID    | Name      | Cluster | Trip Start | Trip Start Time | Trip ID           | Observations | Max Trip Distance (m) | Total Trip Distance (m) | Average Trip Hours | Minimum Trip Hours | Maximum Trip Hours |
| ----- | --------- | ------- | ---------- | --------------- | ----------------- | ------------ | --------------------- | ----------------------- | ------------------ | ------------------ | ------------------ |
| TMC03 | Fire Lily | 1       | 1/27/2016  | 11:00:00        | TMC03_Den1_Trip31 | 7            | 2921                  | 7533                    | 24                 | 21                 | 27                 |
| TMC03 | Fire Lily | 1       | 1/16/2016  | 14:00:00        | TMC03_Den1_Trip17 | 7            | 3003                  | 6052                    | 21                 | 18                 | 24                 |
| TMC03 | Fire Lily | 1       | 1/20/2016  | 11:00:00        | TMC03_Den1_Trip22 | 7            | 1104                  | 3965                    | 21                 | 18                 | 24                 |
| TMC13 | Hope      | 2       | 10/14/2015 | 8:00:00         | TMC13_Den2_Trip4  | 15           | 1414                  | 9071                    | 45                 | 42                 | 48                 |
| TMC13 | Hope      | 2       | 10/12/2015 | 14:00:00        | TMC13_Den2_Trip3  | 12           | 1625                  | 4180                    | 36                 | 33                 | 39                 |
| TMC27 | Disa      | 1       | 9/18/2016  | 8:00:00         | TMC27_Den1_Trip6  | 13           | 3648                  | 11168                   | 69                 | 66                 | 72                 |
| TMC27 | Disa      | 1       | 10/16/2016 | 8:00:00         | TMC27_Den1_Trip26 | 17           | 3667                  | 11249                   | 63                 | 60                 | 66                 |
| TMC28 | Luna      | 1       | 9/21/2016  | 2:00:00         | TMC28_Den1_Trip13 | 8            | 1397                  | 4898                    | 24                 | 21                 | 27                 |
| TMC28 | Luna      | 2       | 10/5/2016  | 2:00:00         | TMC28_Den2_Trip1  | 7            | 695                   | 2964                    | 21                 | 18                 | 24                 |

The 2 FURTHEST (MAX DISTANCE FROM DEN) Trips by caracal:      

​        

| ID     | Name      | Cluster | Trip Start | Trip Start Time | Trip ID           | Observations | Max Trip Distance (m) | Total Trip Distance (m) | Average Trip Hours | Minimum Trip Hours | Maximum Trip Hours |
| ------ | --------- | ------- | ---------- | --------------- | ----------------- | ------------ | --------------------- | ----------------------- | ------------------ | ------------------ | ------------------ |
| TMC03: | Fire Lily | 1       | 1/23/2016  | 11:00:00        | TMC03_Den1_Trip26 | 6            | 3021                  | 7538                    | 18                 | 15                 | 21                 |
| TMC03: | Fire Lily | 1       | 1/16/2016  | 14:00:00        | TMC03_Den1_Trip17 | 7            | 3003                  | 6052                    | 21                 | 18                 | 24                 |
| TMC13: | Hope      | 2       | 10/30/2015 | 5:00:00         | TMC13_Den2_Trip16 | 6            | 2137                  | 4924                    | 18                 | 15                 | 21                 |
| TMC13: | Hope      | 2       | 10/26/2015 | 2:00:00         | TMC13_Den2_Trip13 | 9            | 1951                  | 4399                    | 27                 | 24                 | 30                 |
| TMC27: | Disa      | 1       | 10/16/2016 | 8:00:00         | TMC27_Den1_Trip26 | 17           | 3667                  | 11249                   | 63                 | 60                 | 66                 |
| TMC27: | Disa      | 1       | 9/18/2016  | 8:00:00         | TMC27_Den1_Trip6  | 13           | 3648                  | 11168                   | 69                 | 66                 | 72                 |
| TMC28: | Luna      | 2       | 10/8/2016  | 20:00:00        | TMC28_Den2_Trip5  | 1            | 1731                  | 3398                    | 3                  | 0                  | 6                  |
| TMC28: | Luna      | 3       | 10/16/2016 | 20:00:00        | TMC28_Den3_Trip3  | 1            | 1707                  | 3440                    | 3                  | 0                  | 6                  |

The 2 LARGEST (TOTAL DISTANCE) Trips by caracal:        

| ID    | Name      | Cluster | Trip Start | Trip Start Time | Trip ID           | Observations | Max Trip Distance (m) | Total Trip Distance (m) | Average Trip Hours | Minimum Trip Hours | Maximum Trip Hours |
| ----- | --------- | ------- | ---------- | --------------- | ----------------- | ------------ | --------------------- | ----------------------- | ------------------ | ------------------ | ------------------ |
| TMC03 | Fire Lily | 1       | 1/11/2016  | 11:00:00        | TMC03_Den1_Trip11 | 5            | 2958                  | 7978                    | 15                 | 12                 | 18                 |
| TMC03 | Fire Lily | 1       | 1/23/2016  | 11:00:00        | TMC03_Den1_Trip26 | 6            | 3021                  | 7538                    | 18                 | 15                 | 21                 |
| TMC13 | Hope      | 2       | 10/14/2015 | 8:00:00         | TMC13_Den2_Trip4  | 15           | 1414                  | 9071                    | 45                 | 42                 | 48                 |
| TMC13 | Hope      | 1       | 9/28/2015  | 14:00:00        | TMC13_Den1_Trip4  | 7            | 1654                  | 6379                    | 21                 | 18                 | 24                 |
| TMC27 | Disa      | 1       | 10/16/2016 | 8:00:00         | TMC27_Den1_Trip26 | 17           | 3667                  | 11249                   | 63                 | 60                 | 66                 |
| TMC27 | Disa      | 1       | 9/18/2016  | 8:00:00         | TMC27_Den1_Trip6  | 13           | 3648                  | 11168                   | 69                 | 66                 | 72                 |
| TMC28 | Luna      | 1       | 9/21/2016  | 2:00:00         | TMC28_Den1_Trip13 | 8            | 1397                  | 4898                    | 24                 | 21                 | 27                 |
| TMC28 | Luna      | 1       | 9/20/2016  | 17:00:00        | TMC28_Den1_Trip12 | 2            | 1327                  | 4009                    | 6                  | 3                  | 9                  |

​    

And spearman's rho between denning time and [distance/hours]: this will see if there is a relationship between putative kitten age and trip characteristics: 

| ID    | Name      | Rho (Max Distance) | P-value (Max Distance) | Rho (Total Distance) | P-value (Total Distance) | Rho (Duration Hours) | P-value (Duration Hours) |
| ----- | --------- | ------------------ | ---------------------- | -------------------- | ------------------------ | -------------------- | ------------------------ |
| TMC03 | Fire Lily | 0.158              | 1                      | 0.169                | 1                        | 0.427                | 0.177                    |
| TMC13 | Hope      | 0.214              | 1                      | 0.124                | 1                        | -0.192               | 1                        |
| TMC27 | Disa      | 0.205              | 1                      | 0.296                | 1                        | 0.286                | 1                        |
| TMC28 | Luna      | 0.00509            | 1                      | -0.00902             | 1                        | -0.11                | 1                        |

And results from LM:

​           

| ID    | Name      | Name           | Estimate | Standard Error | Statistic | P Value | Corrected P Value |
| ----- | --------- | -------------- | -------- | -------------- | --------- | ------- | ----------------- |
| TMC03 | Fire Lily | Max Distance   | 0.0232   | 0.0251         | 0.924     | 0.363   | 1                 |
| TMC03 | Fire Lily | Total Distance | 0.0607   | 0.0585         | 1.04      | 0.307   | 1                 |
| TMC03 | Fire Lily | Hours          | 0.307    | 0.125          | 2.46      | 0.0199  | 0.238             |
| TMC13 | Hope      | Max Distance   | 0.00748  | 0.00635        | 1.18      | 0.247   | 1                 |
| TMC13 | Hope      | Total Distance | 0.00559  | 0.0222         | 0.252     | 0.803   | 1                 |
| TMC13 | Hope      | Hours          | -0.102   | 0.116          | -0.875    | 0.388   | 1                 |
| TMC27 | Disa      | Max Distance   | 0.0324   | 0.0219         | 1.48      | 0.15    | 1                 |
| TMC27 | Disa      | Total Distance | 0.0771   | 0.055          | 1.4       | 0.173   | 1                 |
| TMC27 | Disa      | Hours          | 0.295    | 0.283          | 1.04      | 0.307   | 1                 |
| TMC28 | Luna      | Max Distance   | 0.00162  | 0.00506        | 0.321     | 0.75    | 1                 |
| TMC28 | Luna      | Total Distance | -0.00179 | 0.0127         | -0.141    | 0.889   | 1                 |
| TMC28 | Luna      | Hours          | -0.0362  | 0.0489         | -0.741    | 0.462   | 1                 |



# Plot Trips: GIF

```R
# Set basic stuff we need on the cluster rstudio
setwd('C:/Users/herit/My Drive/Research/Caracal/Denning/2023JULY/')
#setwd('~/merondun/misc_research/caracals/')
#.libPaths('~/mambaforge/envs/caracals/lib/R/library')
set.seed(111)
options(scipen = 999)

ibrary(tidyverse)
library(gganimate)
library(ggmap)
register_google(key = "SECRET") # KEEP THIS SECRET

# Define the bounding box of your data
bbox <- st_bbox(st_as_sf(full_trip, coords = c("Longitude", "Latitude"), crs = 4326)) 
names(bbox) = c('left','bottom','right','top')

# Download a satellite map
map <- get_map(bbox, maptype = "satellite", zoom = 11)
ggmap(map)
push = 0.01

#Loop through each ID / den 
for (id in unique(full_trip$ID)) {
  iddat = full_trip %>% filter(ID == id)
  for (clst in unique(iddat$Cluster)) {
    
    cat('Working on ',id,' for den ',clst,'\n')
    #Subset that data 
    dendat = iddat  %>% filter(Cluster == clst)

    den = dendat %>% select(Den_Lat,Den_Long) %>% unique
    # Output gif of the trips 
    # Create a new variable for the color of the lines
    giftrip = dendat %>% mutate(line_color = ifelse(is.na(trip_ID), "white", as.character(trip_ID)))
    
    # Create a acolor map for the trips 
    color_mapping <- setNames(rainbow(n = length(unique(giftrip$trip_ID[!is.na(giftrip$trip_ID)]))), 
                              unique(giftrip$trip_ID[!is.na(giftrip$trip_ID)]))
    color_mapping["white"] <- "white"
    
    # Generate the plot
    p = ggmap(map) +
      # To limit the plot to the bounding box of the current trip:
      coord_cartesian(xlim = c(min(giftrip$Longitude)-push, max(giftrip$Longitude)+push), 
                      ylim = c(min(giftrip$Latitude)-push, max(giftrip$Latitude)+push)) +
      geom_path(data = giftrip, aes(x = Longitude, y = Latitude, color = line_color, group = trip_ID), size = 1) +
      geom_point(data = den, aes(x = Den_Long, y = Den_Lat), inherit.aes = FALSE, pch = 21, size = 5, fill = 'white') +  
      scale_color_manual(values = color_mapping) +
      theme_minimal()+
      theme(legend.position='none')
    
    # Add time transitions
    animation = p + 
      transition_reveal(Date_time) +
      labs(title = paste0(id,': ',clst), x = "Longitude", y = "Latitude") +
      ease_aes('linear')
    
    # Render the animation
    animate(animation, renderer = gifski_renderer(paste0(id,'_',clst,'.gif')))

  }
}

```

