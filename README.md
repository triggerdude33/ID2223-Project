# ID2223-Project

Project page is live and can be viewed at: [https://triggerdude33.github.io/ID2223-Project/](https://triggerdude33.github.io/ID2223-Project/)

## Introduction

The aim of the project is to predict alpine ski resort closures for the next 15 years due to climate change. 

### Prediction problem

Predict, for all current open ski resorts in the alps, if it will close down due to climate change in more than 15 years or before then. If before 15 years from now, specify which year.

### Data sources
We have the following dynamic data sources:
* [Open-meteo](https://open-meteo.com)
* [Abandoned ski towns](https://abandonedskitowns.com/)
* [Open street map](https://www.openstreetmap.org)

### Assumptions and limitations taken regarding the data

* We assume that all ski resorts which have closed down, did so due to climate change.
* We only look at winter season weekly weather (november-march) for predicting ski resort closure.
  * No daily temperature, snow depth or wind etc.

## Method

### Overview

The project is divided into 5 steps. See the table below.
| Step | input | process | output |
| --- | --- | --- | --- |
| 1_annual_resort_feature_pipeline | [Abandoned ski towns](https://abandonedskitowns.com/) and [Open street map](https://www.openstreetmap.org) | from sources, prepares ski resort name, latitude, longitude and id for export to hopsworks | `current_resorts` and `former_resorts` fg
| 2_annual_weather_feature_pipeline | `current_resorts`, `former_resorts`, open-meteo | get historical weather data for open and closed resorts | `ski_weather` fg
| 3_weather_model | `current_resorts`, `ski_weather` fg | creates predictions for open resorts future winter climate | `predicted_ski_weather` fg |
| 4_shutdown_model | `closed_resorts`, `open_resorts`, `ski_weather`, `predicted_ski_weather` feature groups | trains on historical data and creates predictions for when currently open ski resorts will close down | `shutdown_predictions` fg |
| 5_update_dashboard | `shutdown_predictions` fg | updates the table on the dashboard with the latest added ski resort shutdown predictions | [github page](https://triggerdude33.github.io/ID2223-Project/)
: Table fg stands for "feature group"

## Technologies used




## Discussion

If we were to input more dimensions to the shutdown model such as snow depth and wind, it will probably achieve a higher prediction rate. This would require additions to the weather_model, it would need to generate weather data for these dimensions also. 

#### Abandoned ski towns

* We assume that every ski resort closure registered occured due to climate change i.e lack of snow
* We take no regard to what year ski resorts open, we only utilize the year which they close.
* We take resorts from north America also. This was due to there not being enough qualified closed ski resort samples in Europe to create a large enough training dataset.
