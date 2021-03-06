Forecasting Metadata
================

``` r
library(EML)
```

    ## Warning: package 'EML' was built under R version 3.5.2

``` r
library(tidyverse)
```

    ## Warning: package 'tidyverse' was built under R version 3.5.2

    ## Warning: package 'ggplot2' was built under R version 3.5.2

    ## Warning: package 'tidyr' was built under R version 3.5.2

    ## Warning: package 'purrr' was built under R version 3.5.2

    ## Warning: package 'dplyr' was built under R version 3.5.2

    ## Warning: package 'stringr' was built under R version 3.5.2

    ## Warning: package 'forcats' was built under R version 3.5.2

``` r
library(uuid)
emld::eml_version("eml-2.2.0")
```

    ## [1] "eml-2.2.0"

``` r
set.seed(42)
```

A simple, example forecast of population Growth of two interacting species

========================================================

First, set the forecast identifiers

ForecastProject\_id represents the launch of an automated, iterative forecast. It is created each time a human modifies the forecast code. It can be a DOI because this is the level that we envision citations occuring.

Forecast\_id represents each forecast cycle within a ForecastProject\_id

For example, if you have a forecast code base on GitHub and launch a forecast from that code that runs daily for 365 days, then there will be one ForecastProject\_id and 365 Forecast\_ids. A paper analyzing the forecasts would cite the ForecastProject\_id.

``` r
forecast_issue_time <- as.Date("2001-03-04")
#Forecast_id <- uuid::UUIDgenerate() #ID that applies to the specific forecast
Forecast_id <- "20010304T060000"     # ISO datetime should make a valid Forecast_id
ForecastProject_id <- 30405043 #Some ID that applies to a set of forecasts
```

Generating the forecast
-----------------------

Multi-species growth for multiple depths with process uncertainty

``` r
NT <- 30
n_ensembles <- 10
n_depths <- 3
depths <- c(1, 3, 5)

r <- c(1, 3)
K <-  c(10, 20)
alpha <-  c(0.2, 0.3)
n0 <-  0.5

n <- array(NA,dim = c(2, NT, n_depths, n_ensembles))
n[,1,,] <- n0

process_sd <- 0.01

data_assimilation <- rep(0, NT)  

for(t in 2:NT){
  data_assimilation[t] <- 0
  for(depth in 1:n_depths){
    for(ens in 1:n_ensembles){
      n[1, t, depth, ens] <-  n[1, t-1, depth, ens] + 
          r[1]*n[1, t-1,depth,ens]*(1-((n[1, t-1, depth, ens] + 
          alpha[1]*n[2, t-1, depth, ens])/K[1])) + rnorm(1, 0, process_sd)
      n[2, t, depth, ens] <-  n[2, t-1, depth, ens] + 
          r[2]*n[2, t-1, depth, ens]*(1-((n[2, t-1, depth, ens] + 
          alpha[2]*n[1, t-1, depth, ens])/K[2])) + rnorm(1, 0, process_sd)
    }
  }
}
```

Saving to a standardized output format
--------------------------------------

Standard Option 1: netCDF
=========================

Convert to a netcdf format

``` r
library(ncdf4)
```

    ## Warning: package 'ncdf4' was built under R version 3.5.2

``` r
ncfname <- "logistic-forecast-ensemble-multi-variable-space-long.nc"

time <- as.Date(as.character(2000 + 1:NT), format = "%Y")

data_assimilation <- rep(0, length(time))

#Set dimensions
ens <- as.integer(seq(1,n_ensembles,1))
depths <- as.integer(c(1,2,3))
timestep <- as.integer(seq(1, NT, 1))

ensdim <- ncdim_def("ens", 
                    units = "",
                    vals = ens, 
                    longname = 'ensemble member') 
depthdim <- ncdim_def("depth", 
                      units = "meters",
                      vals = depths, 
                      longname = 'Depth from surface') 
timedim <- ncdim_def("timestep", 
                     units = '1 day', 
                     longname = 'timestep',
                     vals = timestep)

dimnchar   <- ncdim_def("nchar",   "", 
                        1:nchar(as.character(time[1])), 
                        create_dimvar=FALSE)

#Define variables
fillvalue <- 1e32

def_list <- list()
def_list[[1]] <- ncvar_def(name = "time",
                           units = "datetime",
                           dim = list(dimnchar, timedim),
                           longname = "time",
                           prec="char")
def_list[[2]] <- ncvar_def(name =  "species_1",
                           units = "number of individuals",
                           dim = list(timedim, depthdim, ensdim),
                           missval = fillvalue,
                           longname = 'temperature_mean',
                           prec="single")
def_list[[3]] <- ncvar_def(name =  "species_2",
                           units = "number of individuals",
                           dim = list(timedim, depthdim, ensdim),
                           missval = fillvalue,
                           longname = 'temperature_mean',
                           prec="single")
def_list[[4]] <- ncvar_def(name =  "data_assimilation",
                           units = "logical",
                           dim = list(timedim),
                           missval = fillvalue,
                           longname = '1 = data assimilation used in timestep',
                           prec="single")

ncout <- nc_create(ncfname,def_list,force_v4=T)

ncvar_put(ncout,def_list[[1]] , time)
ncvar_put(ncout,def_list[[2]] , n[1, , , ])
ncvar_put(ncout,def_list[[3]] , n[2, , , ])
ncvar_put(ncout,def_list[[4]] , data_assimilation)

#Global file metadata
ncatt_put(ncout,0,"ForecastProject_id", as.character(ForecastProject_id), 
          prec =  "text")
ncatt_put(ncout,0,"Forecast_id",as.character(Forecast_id), 
          prec =  "text")
ncatt_put(ncout,0,"forecast_issue_time",as.character(forecast_issue_time), 
          prec =  "text")
nc_close(ncout)
```

Standard Option 2: ensemble CSV
===============================

Convert to a flat file format (CSV) with one column for each variable and all ensemble members saved

``` r
time <- as.Date(as.character(2000 + 1:NT), format = "%Y")
state_names <- c("species_1", "species_2")
n_states <- length(state_names)
states <- list(n[1, , , ], n[2, , ,])

#No data assimilation was used for any of forecast results archived

df_combined <- list()

for(k in 1:n_states){
  for(i in 1:n_depths){
    df <- as_tibble(states[[k]][, ,i])
    names(df) <- as.character(seq(1, ncol(states[[k]][, ,i])))
    df <- cbind(time, df, data_assimilation)
    df <- df %>% 
      pivot_longer(cols = -c(time,data_assimilation), 
                   names_to = "ensemble", 
                   values_to = state_names[k]) %>% 
      mutate(ensemble = as.integer(ensemble)) %>% 
      mutate(depth = depths[i])
    if(i == 1){
      running_df <- df
    }else{
      running_df <- rbind(running_df, df)
    }
  }
  df_combined[[k]] <- running_df
}
```

    ## Warning: The `x` argument of `as_tibble.matrix()` must have column names if `.name_repair` is omitted as of tibble 2.0.0.
    ## Using compatibility `.name_repair`.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_warnings()` to see where this warning was generated.

``` r
df_combined <- right_join(df_combined[[1]], df_combined[[2]], 
                          by = c("time", "ensemble", "depth", "data_assimilation")) %>% 
  mutate(forecast_issue_time = forecast_issue_time,
         Forecast_id = Forecast_id,
         ForecastProject_id = ForecastProject_id) %>% 
  select(time, depth, ensemble, state_names[1], 
         state_names[2], forecast_issue_time, 
         data_assimilation, ForecastProject_id, Forecast_id) 

df_combined
```

    ## # A tibble: 270 x 9
    ##    time       depth ensemble species_1 species_2 forecast_issue_…
    ##    <date>     <int>    <int>     <dbl>     <dbl> <date>          
    ##  1 2001-04-24     1        1     0.5        0.5  2001-03-04      
    ##  2 2001-04-24     1        2     0.5        0.5  2001-03-04      
    ##  3 2001-04-24     1        3     0.5        0.5  2001-03-04      
    ##  4 2002-04-24     1        1     0.984      1.95 2001-03-04      
    ##  5 2002-04-24     1        2     0.967      1.93 2001-03-04      
    ##  6 2002-04-24     1        3     0.972      1.95 2001-03-04      
    ##  7 2003-04-24     1        1     1.83       7.13 2001-03-04      
    ##  8 2003-04-24     1        2     1.82       7.09 2001-03-04      
    ##  9 2003-04-24     1        3     1.82       7.15 2001-03-04      
    ## 10 2004-04-24     1        1     3.05      20.3  2001-03-04      
    ## # … with 260 more rows, and 3 more variables: data_assimilation <dbl>,
    ## #   ForecastProject_id <dbl>, Forecast_id <chr>

``` r
write.csv(df_combined, 
          file = "logistic-forecast-ensemble-multi-variable-multi-depth.csv")
```

Standard Option 3: summary CSV
==============================

Convert to a flat file format (CSV) with forecast distribution summaries saved

``` r
df_species_1 <- df_combined %>% 
  select(-species_2) %>% 
  group_by(time, depth, forecast_issue_time, data_assimilation, 
           ForecastProject_id, Forecast_id) %>% 
  summarize(mean = mean(species_1),
            Conf_interv_02.5 = quantile(species_1, 0.025),
            Conf_interv_97.5 = quantile(species_1, 0.975)) %>% 
  pivot_longer(cols = c("mean","Conf_interv_02.5","Conf_interv_97.5"),
               names_to = "Statistic",
               values_to = "species_1")

df_species_2 <- df_combined %>% 
  select(-species_1) %>% 
  group_by(time, depth, forecast_issue_time, data_assimilation, 
           ForecastProject_id, Forecast_id) %>% 
  summarize(mean = mean(species_2),
            Conf_interv_02.5 = quantile(species_2, 0.025),
            Conf_interv_97.5 = quantile(species_2, 0.975)) %>% 
  pivot_longer(cols = c("mean","Conf_interv_02.5","Conf_interv_97.5"),
               names_to = "Statistic",
               values_to = "species_2")

 df_summary <- right_join(df_species_1, df_species_2)
```

    ## Joining, by = c("time", "depth", "forecast_issue_time", "data_assimilation", "ForecastProject_id", "Forecast_id", "Statistic")

``` r
 df_summary
```

    ## # A tibble: 270 x 9
    ## # Groups:   time, depth, forecast_issue_time, data_assimilation,
    ## #   ForecastProject_id [90]
    ##    time       depth forecast_issue_… data_assimilati… ForecastProject…
    ##    <date>     <int> <date>                      <dbl>            <dbl>
    ##  1 2001-04-24     1 2001-03-04                      0         30405043
    ##  2 2001-04-24     1 2001-03-04                      0         30405043
    ##  3 2001-04-24     1 2001-03-04                      0         30405043
    ##  4 2001-04-24     2 2001-03-04                      0         30405043
    ##  5 2001-04-24     2 2001-03-04                      0         30405043
    ##  6 2001-04-24     2 2001-03-04                      0         30405043
    ##  7 2001-04-24     3 2001-03-04                      0         30405043
    ##  8 2001-04-24     3 2001-03-04                      0         30405043
    ##  9 2001-04-24     3 2001-03-04                      0         30405043
    ## 10 2002-04-24     1 2001-03-04                      0         30405043
    ## # … with 260 more rows, and 4 more variables: Forecast_id <chr>,
    ## #   Statistic <chr>, species_1 <dbl>, species_2 <dbl>

``` r
write.csv(df_summary, 
          file = "logistic-forecast-summary-multi-variable-multi-depth.csv")
```

Standardized Metadata
---------------------

Let's document the metadata of the data table itself. It may well be that we decide an Ecological Forecast has to have specific columns like the ones described above, which would thus correspond to a partially pre-defined attributes table (e.g. the units would probably still be allowed to vary, but format would be the same.)

Note one weakness of this format is that it assumes all data in a column have the same units. This common assumption might be violoated by transformations to "long" form data, where you have columns like "variable", "value", and "units". (The long form may be useful, but it exposes much less information in the metadata layer -- e.g. we no longer know what's actually being measured without looking at the data file itself).

``` r
attributes <- tibble::tribble(
  ~attributeName, ~attributeDefinition, ~unit, ~formatString, ~numberType, ~definition,
  "time",          "time",                       "year",     "YYYY-MM-DD", "numberType", NA,
  "depth",         "depth in reservior",         "meter",   NA,          "real", NA,
  "ensemble",      "index of ensemble member",   "dimensionless",    NA,         "integer", NA,
  "species_1",     "Population size of species 1", "numberPerMeterSquared", NA,  "real", NA,
  "species_2",     "Population size of species 2", "numberPerMeterSquared", NA,  "real", NA,
  "forecast_issue_time",     "time that forecast was created", NA, "YYYY-MM-DD",  NA, NA,
  "data_assimilation",     "Flag whether time step included data assimilation", "dimensionless", NA, "integer", NA,
  "Forecast_id",     "ID for specific forecast cycle", NA, NA,  NA, "forecast id",
  "ForecastProject_id",     "ID for forecasting project", NA, NA,  NA, "project id"
)
attrList <- set_attributes(attributes, 
                           col_classes = c("Date", "numeric", "numeric", 
                                           "numeric","numeric", "Date",
                                           "numeric", "character", "character"))
physical <- set_physical("logistic-forecast-ensemble-multi-variable-multi-depth.csv")
```

    ## Automatically calculated file size using file.size("logistic-forecast-ensemble-multi-variable-multi-depth.csv")

    ## Automatically calculated authentication size using digest::digest("logistic-forecast-ensemble-multi-variable-multi-depth.csv", algo = "md5", file = TRUE)

``` r
dataTable <- eml$dataTable(
                 entityName = "logistic-forecast-ensemble-multi-variable-multi-depth.csv",
                 entityDescription = "Forecast of population size using a depth specific model",
                 physical = physical,
                 attributeList = attrList)
```

There's a lot more optional terminology that could be exploited here -- for instance, the specification lets us define different missing value codes (and explanations) for each column, and allows us to indicate `precision`, `minimum` and `maximum`.

Note that `physical` type can document almost any formats as well, including NetCDF etc. A NetCDF file would still document the variables measured in much the same way regardless of the underlying representation. Note that

Now that we've documented the actual data.frame itself, we can add additional metadata to the record describing our forecast, which is essential for citing, discovering, and interpreting the result. We start with some authorship information.

``` r
me <- list(individualName = list(givenName = "Quinn", 
                                 surName = "Thomas"),
           electronicMailAddress = "rqthomas@vt.edu",
           id = "https://orcid.org/0000-0003-1282-7825")
```

Set Taxonomic, Temporal, and Geographic Coverage. (Look, apparently we're modeling population densities of *Sarracenia purpurea* in Harvard Forest starting in about 2012!)

``` r
coverage <- 
  set_coverage(begin = '2012-06-01', 
               end = '2013-12-31',
               sci_names = "Sarracenia purpurea",
               geographicDescription = "Harvard Forest Greenhouse, Tom Swamp Tract (Harvard Forest)",
               west = -122.44, east = -117.15, 
               north = 37.38, south = 30.00,
               altitudeMin = 160, altitudeMaximum = 330,
               altitudeUnits = "meter")
```

Set key words. We will need to develop a EFI controlled vocabulary

``` r
keywordSet <- list(
    list(
        keywordThesaurus = "EFI controlled vocabulary",
        keyword = list("forecast",
                    "population",
                    "timeseries")
    ))
```

Our dataset needs an abstract describing what this is all about. Also, a methods section is not required but it's probably a good idea.

We envision having additional required metadata sections. However, we need to figure out the formatting of the Markdown file so that it is represented cleanly in the EML.

``` r
additionalMetadata <- eml$additionalMetadata(
  #  describes="forecast",  ## not sure how to find the correct ID for this to be valid
  metadata = list(
    forecast = list(
      timestep = "1 year", ## should be udunits parsable; already in coverage -> temporalCoverage?
      forecast_horizon = "30 years",
      initial_conditions = list(
        # Possible values: no, contains, data_driven, propagates, assimilates
        uncertainty = "contains",
        # Number of parameters / dimensionality
        complexity = 2
      ),
      parameters = list(
        uncertainty = "contains",
        complexity = 3
      ),
      random_effects = list(
        uncertainty = "no"
      ),
      process_error = list(
        uncertainty = "propagates",
        propagation = list(
          type = "ensemble", # ensemble vs analytic
          size = 10          # required if ensemble
        ),
        complexity = 1
      ),
      drivers = list(
        uncertainty = "no"
      )
      # assimilation_method ## required if any uncertainty = assimilates
    ) # forecast
  ) # metadata
) # eml$additionalMetadata
```

**Model Description**

-   Type of model (Empirical, process-based, machine learning): Process-based
-   Model name: discrete Lotka–Volterra model
-   Location of repository with model code: <https://github.com/somewhere> or <https://doi.org/10.xxx>
-   Model citation: N/A
-   Total number of model process parameters: 3

**Model Covariates**

-   Type (i.e., meteorology): N/A
-   Source (i.e., NOAA GEFS): N/A

``` r
abstract <- list(markdown = paste(readLines("abstract.md"), collapse = "\n"))
```

    ## Warning in readLines("abstract.md"): incomplete final line found on
    ## 'abstract.md'

``` r
methods <- list(id="forecast",methodStep = list(description = list(markdown = paste(readLines("methods.md"), collapse = "\n"))))  ## to be dropped
```

``` r
dataset = eml$dataset(
               title = "A very silly logistic forecast",
               creator = me,
               contact = list(references="https://orcid.org/0000-0003-1282-7825"),
               pubDate = forecast_issue_time,
               intellectualRights = "http://www.lternet.edu/data/netpolicy.html.",
               abstract =  "An illustration of how we might use EML metadata to describe an ecological forecast",
               dataTable = dataTable,
               keywordSet = keywordSet,
               coverage = coverage,
               methods = methods
               )
```

All we need now is to add a unique identifier for the project and we are good to go! This could be a DOI or merely an identifier we create, e.g. a UUID.

``` r
my_eml <- eml$eml(dataset = dataset,
           additionalMetadata = additionalMetadata,
           packageId = Forecast_id,  #Is this the ForecastProject_ID?
           #What about the Forecast_ID
           system = "uuid"
           )
```

Once we have finished building our EML metadata, we can confirm it is valid.
This will catch any missing elements. (Recall that what is 'required' depends on what you include -- for example, you don't have to document a `dataTable` at all, but if you do, you have to document the "physical" file format it is in
(e.g. `csv`) and the attributes and units it uses!)

``` r
eml_validate(my_eml)
```

    ## [1] TRUE
    ## attr(,"errors")
    ## character(0)

We are now ready to write out a valid EML document:

``` r
write_eml(my_eml, "forecast-eml.xml")
```

    ## NULL

At this point, we could easily upload this metadata along with the data itself to DataONE via the API (or `dataone` R package.)

We can also generate a JSON-LD version of EML:

``` r
emld::as_json(as_emld("forecast-eml.xml"), file = "forecast-eml.json")
```
