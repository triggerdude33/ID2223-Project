Project name: Predicting Alpine ski resort closures for the next 15 years due to climate change.  
Group members: Emil Lidbom and Matteus Berg

**Dynamic data sources**

* [Open-meteo](https://open-meteo.com/) \- Will be used to get historical data regarding snow depth and temperature for ski resort locations, both for former ski resorts and current.  
* [Abandoned ski towns](https://abandonedskitowns.com/%20) \- Will be used to find the year and coordinates in which different former alpine ski resorts closed for good.   
  * Lists about 500 ski resorts in Europe and North America which have closed down in the last \~60 years. For each resort, it lists the coordinates of the resort, what year the resort opened and what year it closed down.  
  * Contains backend API which can be called directly without the need to visit the website.  
* [Open street map](https://www.openstreetmap.org%20) \- Will be used to find coordinates for currently operational alpine ski resorts  
  * The [Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API/Areas) will be used to fetch all “winter sports Areas” in the European alpine region.  
  * Contains 442 open ski resorts in the alps

**The prediction problem that you want to solve**

**Problem specification.** Predict, for all current open ski resorts in the alps, if it will close down due to climate change in more than 15 years or before then. If before 15 years from now, specify which year.

We will create the following feature groups in hopsworks:

* ski\_weather  
  * resort\_id, temperature, snow depth and coordinates   
  * Snow depth and temperature for every day the last 15 years during the winter half year (november-march). For current ski resort locations.   
  * For former resorts, 15 years before and respectively after the resort closed will be added.   
  * In total, this should be about 20 MB of data.  
* former\_resorts  
  * resort\_id, name and coordinates, opening year and closing year for all former ski resorts.  
* open\_resorts  
  * resort\_id, name and coordinates for all currently open resorts

**Regarding models.** Two different models will be created. Weather\_model and shutdown\_model. Weather\_model will predict future temperatures and snow depth based on historical weather data. Shutdown\_model will, given a time interval of ski\_weather data, estimate on which year the ski resort closed/will close down. 

**Regarding training.** 

* The weather\_model will receive the first 15 years of a ski\_weather data item (daily snow depth, temperature) for a former resort. It is then tasked to predict the daily snow depth and temperature for the next 15 years.  
* The shutdown\_model will receive a randomly picked 15 year slice of the 30 year total interval for a specific ski\_weather data item (daily snow depth, temperature) belonging to a former resort. It is then, based on the daily temperature and snow depth numbers, tasked to predict what year the resort shut down for business.

**Regarding inference.** For an item in current\_resort, weather\_model will predict the temperature and snow depth for the next 15 years, using the last 15 years as input. After that, shutdown\_model will predict, in the future 15 year interval, what year the resort should shut down. The inference for each resort will be a separate github action. The inference jobs will be executed on open\_resorts in alphabetical order.

**The UI you will provide to show the value of the predictions**

We will display a table of all open alpine ski resorts. For each resort, if shutdown\_model predicts the resort to close in less than 15 years, that year will be displayed. Otherwise “\>15 years away” will be displayed. The UI will be a github page. New rows to the table will be added when github actions for inference jobs get completed.

**An overview of the technologies you will use**

We will use

* Hopsworks Feature Store to store feature groups  
* Hopsworks as our model registry. To store the models which will make predictions  
* GitHub actions as our compute platform. Will execute training pipelines on our models and batch inference, among other things.  
* GitHub pages as our serverless UI

**Att göra**

* Rensa bort all data för stängda anläggningar som inte stängdes på grund av brist på snö.  
* Ta med tex. Vind. Kanske extremväder också påverkar viljan att åka skidor på en ort?  
* För stängda skidorter. Vissa har endast en decimals nogranhet på latitud och longitud. Ta bort dessa orter eftersom koordinaterna inte är tillräckligt exakta?

