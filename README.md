# ID2223-Project

Project page is live and can be viewed at: [https://triggerdude33.github.io/ID2223-Project/](https://triggerdude33.github.io/ID2223-Project/)

## Introduction

The aim of the project is to predict ski resort closures in the European alps for the next 15 years due to climate change.

### Prediction problem

Predict, for all current open ski resorts in the alps, if it will close down due to climate change in more than 15 years or before then. If before 15 years from now, specify which year.

### Solution overview

To solve the above prediction problem, we have implemented two models:
* `weather_model`. Given 30 years of historical weekly temperature data, predict the future 15 years of weekly temperature data.
* `shutdown_model`. Given a 15-year time interval of weekly temperature data, estimate on which year the ski resort closed/will close down. 

### Data sources
We have the following dynamic data sources:
* [Open-meteo](https://open-meteo.com)
* [Abandoned ski towns](https://abandonedskitowns.com/)
* [Open street map](https://www.openstreetmap.org)

### Assumptions and limitations taken regarding the data

* We assume that all ski resorts which have closed down, did so due to climate change.
* We only look at winter season weekly weather (november-march) for predicting ski resort closure.
  * No daily temperature, snow depth or wind etc.
* We take resorts from North America also to train `shutdown_model`. This was due to there not being enough qualified closed ski resort samples in Europe to create a large enough training dataset.

## Method

### Overview

The project is divided into 5 steps. See the table below.
| Step | input | process | output |
| --- | --- | --- | --- |
| 1: annual resort feature pipeline | [Abandoned ski towns](https://abandonedskitowns.com/) and [Open street map](https://www.openstreetmap.org) | from sources, prepares ski resort name, latitude, longitude and id for export to hopsworks | `current_resorts` and `former_resorts` fg
| 2: annual weather feature pipeline | `current_resorts`, `former_resorts` fg, [Open-meteo](https://open-meteo.com) | get historical weather data for open and closed resorts | `ski_weather` fg
| 3: `weather_model` | `current_resorts`, `ski_weather` fg | creates predictions for open resorts future winter climate | `predicted_ski_weather` fg |
| 4: `shutdown_model` | `closed_resorts`, `open_resorts`, `ski_weather`, `predicted_ski_weather` fg | trains on historical data and creates predictions for when currently open ski resorts will close down | `shutdown_predictions` fg |
| 5: update dashboard | `shutdown_predictions` fg | updates the table on the dashboard with the latest added ski resort shutdown predictions | [github page](https://triggerdude33.github.io/ID2223-Project/)

Note. "fg" stands for "feature group"

Each step corresponds to a python notebook in the github repository.

The goal was to have each step run as a scheduled pipeline on github. This to get an automatic update for future ski resort closures, based on the data fetched from the data sources. However, we were unable to reach this state. A large problem was the API limits for Open-Meteo, more on this is described in Step 2.

### Step 1: Annual resort feature pipeline

Firstly, and API requesst is sent to the backend database of [Abandoned ski towns](https://abandonedskitowns.com/). From this, data for all closed down resorts are obtained. We receive name, latitude, longitude, and year which the resort closed down. Many resorts are filtered out due to two reasons:
* The resort does not list the exact year closure
* The resort is not located in Europe or North America

Once the filtering is done, the remaining data is uploaded to hopsworks. Open resorts are uploaded to `current_resorts` and closed resorts are uploaded to `former_resorts`.

Below are some statistics:
* 241 closed resorts were uploaded to hopsworks
* 839 open resorts were uploaded to hopsworks

### Step 2: Annual weather feature pipeline

#### Open-Meteo API limits

There is a 600 API call limit per minute to open meteo. The number of API calls corresponds to the number of dates you query for. Furthermore, you can query for the same date for several locations, and it will scale less than lineraly seen to the amount of API calls. This is why we decided to fetch temperatures on a date basis, instead of on a ski resort location basis

#### Fetching data

In this step, all weather data for either closed or open resorts for a specific year is fetched. The `ski_weather` fg is downloaded. It is checked what latest year `ski_weather` contains, firstly for closed resorts and then for open resorts. Name the latest year `y`. If `y` for any resort type is found to be smaller than last year, it will download temperature data for `y+1` from Open-Meteo for all resorts for that resort type (for open resorts only data for the first 100 resorts are downloaded, see discussion paragraph below). The data received from Open-Meteo will be of the average daily temperature for the winter season months (November-March). Only fetching weather data for winter season months (November-March) was advantageous for saving API calls to Open-Meteo. We believe fetching temperatures for the winter months would add very little to our prediction performance. The daily temperature data is reformatted to weekly temperature data. The weekly temperature is calculated by taking the mean of all days making up that week. The data is then uploaded to hopsworks.


#### Differences in fetching for open and closed resorts

For closed resorts, weather data for 15 years before and 15 years after the resort closing is fetched. This gives us a 30 year interval of historical weather data in total. This is done to better train `shutdown_model`: For `shutdown_model` traning, we want to select a random 15 year interval of the 30 year total, then make `shutdown_model` attempt to predict what year the resort closed down. This way, we may for a training sample vary which year a resort closes down, so that `shutdown_model` for its predictions will fit to temperatures instead of specific years in the interval. More on this in Step 4.

For open resorts, weather data for the last 30 years is fetched. For the purposes of `shutdown_model`, only the last 15 years are necessary, since `weather_model` will predict the last 15. For `weather_model` however, we found that 15 years of historiccal data seemed a bit to small for seeing any substantial climate warming trend. Thus, we increased from 15 to 30 years of historical data for open resort, so that `weather_model` may better optimize its **Trend kernel** (See "Step 4 `weather_model`").

#### Discussion

We were not able to fetch temperature data for all resorts at once from Open-Meteo, due to the API limit. So the decision was made to instead fetch temperature data first for closed resorts and then for open resorts, one winter year at a time. This worked for closed resorts, but for open resorts we hit the API limit once again. To solve this, we only fetched temperatures for the first 100 open resorts and skipped predicting shutdown for the other open resorts. Had there been more time, we would have modified the notebook so that it fetches all temperature data for a single year for a particular resort `r`. Then the notebook could be run on an hourly pipeline, meaning we would get all necessary data for an open resort after about 15 hours. We could then have had `weather_model` and `shutdown_model` run on a daily pipeline, predicting weather values and shutdown year for `r`.

### Step 3: `weather_model`

To predict future weather for ski resorts, we utilize a gaussian process model. The model has three different kernels, meant to model different aspects of the weather:
* **Trend kernel**. This kernel models the climate change. It is set to force the trend to be linear. That is, each year it will increase the temperature by the same amount. 
* **Seasonal kernel**. Models seasonal variety in temperature. It is a periodic kernel, where the period is one year. 
* **Weather kernel**. Models the unpredictable weather chaos.
The final predicted result for a week is the sum of the three kernels.

To say the least, it is a bold statement to assume that climate change is linear. It could also be exponential, or perhaps polynomial. Had there been more time, we would have explored different trend kernels to model the climate trend. The different growth expectations would need to be compared using some form of metric. Thus a fitting prediction performance metric would need to be found also.

`weather_model` is not trained on historical data for other ski resorts before making its inference. In other words, it only optimizes its hyperparameters using the historical data for the current resort it is predicting for. Temperature trends and seasonal variety differs between locations, therefore we don't see the value in training the model on closed resort weather before making inference on open resort weather. In addition, we haven't found any examples of gaussian process models being trained in this way: If traning on closed resorts would be considered, a metric would need to be found which could measure the accuracy of `weather_model` temperature predictions against the actual historical data. 

For a resort, `weather_model` outpus as many years of predicted data as the resort has historical data in hopsworks. This design decision was done simply because it saved on development time. As explained in Step 2, 30 years of historical data is fetched for open resorts. This means that `weather_model` predicts 30 years of data for each open resorts and then uploads it onto hopsworks. Thus, 15 years of unnecessary data is uploaded onto hopsworks, since `shutdown_model` will only use the first 15 future years, not the later 15. Had there been more time, we would have made the amount of years `weather_model` should predict into a variable which could be set at the beginning of the notebook.

See the two below figures for a visualization of a typical `weather_model` prediction for a ski resort. 

<img src="img/wm_weekly_temperature.png" alt="weekly temperature picture not loaded" width="600"/>

<img src="img/wm_yearly_temperature.png" alt="yearly temperature picture not loaded" width="600"/>

Note that for the yearly temperatures figure, it can be observed that the identical week from year to year increases by the same constant amount. This shouldn't be the case, **Weather kernel** should create slight weather chaos to the temperature values.  Hence, it could imply that the **Weather kernel** wasn't correctly setup.


### Step 4: `shutdown_model`

#### Discussion

If we were to input more dimensions to the `shutdown_model` such as snow depth and wind, it will probably achieve a higher prediction rate. This would require additions to the weather_model, it would need to generate weather data for these dimensions also. 

### Step 5: Update dashboard

This step fetches all data rows from the `shutdown_predictions` feature group. It then dynamically creates a markdown table listing the data. The file `docs/index.md` in the github repository is then updated with the new markdown table. A daily pipeline schedule is currently running. It executes this step and commits the file changes made in `docs/index.md` to the github repository. This way, the dashboard [github page](https://triggerdude33.github.io/ID2223-Project/) is automatically updated.



