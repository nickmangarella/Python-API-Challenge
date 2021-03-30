# weather-api-stats

## Description
Used the OpenWeatherMap API, Google Maps API, Pandas, NumPy, MatPlotLib, and SciPy Statistics to create and analyze multiple vizualizations of the weather data on a given day for 500+ randomly selected cities from around the world.

## WeatherPy
```
# Dependencies and Setup
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import requests
import time
from scipy.stats import linregress

# Import API key
from config import weather_key

# Incorporated citipy to determine city based on latitude and longitude
from citipy import citipy

# Output file path
output_file = "output_data/cities.csv"

# Range of latitudes and longitudes
lat_range = (-90, 90)
lng_range = (-180, 180)
```

### Generate Cities List
```
# List for holding lat_lngs and cities
lat_lngs = []
cities = []

# Create a set of random lat and lng combinations
lats = np.random.uniform(lat_range[0], lat_range[1], size=1500)
lngs = np.random.uniform(lng_range[0], lng_range[1], size=1500)
lat_lngs = zip(lats, lngs)

# Identify nearest city for each lat, lng combination
for lat_lng in lat_lngs:
    city = citipy.nearest_city(lat_lng[0], lat_lng[1]).city_name
    
    # If the city is unique, then add it to a our cities list
    if city not in cities:
        cities.append(city)

# Print the city count to confirm sufficient count
len(cities)
```

### Perform API Calls
```
# Built url for API calls
url = "http://api.openweathermap.org/data/2.5/weather?"
units = "metric"
query_url = f"{url}appid={weather_key}&units={units}&q="

# Lists to hold city data
number = int()
city_name = []
lat = []
lng = []
max_temp = []
humidity = []
clouds = []
wind_speed = []
country = []
date = []

# For loop to grab city weather data with weather API
print(f"Beginning Data Retrieval")
print(f"--------------------")
for city in cities:
    number = number + 1
    
    # Exception to run through each city, collecting data and passing cities not found
    try:
        response = requests.get(query_url + city).json()
        city_name.append(response['name'])
        lat.append(response['coord']['lat'])
        lng.append(response['coord']['lon'])
        max_temp.append(response['main']['temp_max'])
        humidity.append(response['main']['humidity'])
        clouds.append(response['clouds']['all'])
        wind_speed.append(response['wind']['speed'])
        country.append(response['sys']['country'])
        date.append(response['dt'])
        print(f"Processing Record {number} | {city}")
    except:
        print(f"City not found. Skipping...")
        pass
print(f"--------------------")
print(f"Data Retrieval Complete")
print(f"--------------------")
```
```
# Create a dataframe and add city weather data
city_data = pd.DataFrame()

city_data['City'] = city_name
city_data['Lat'] = lat
city_data['Lng'] = lng
city_data['Max Temp'] = max_temp
city_data['Humidity'] = humidity
city_data['Cloudiness'] = clouds
city_data['Wind Speed'] = wind_speed
city_data['Country'] = country
city_data['Date'] = date

# Export city data to a csv
city_data.to_csv(output_file, encoding="utf-8", index=False)

# Visualize
city_data.head()
```
```
# Describe function to return city data stats
city_stats = city_data[["Lat", "Lng", "Max Temp", "Humidity", "Cloudiness", "Wind Speed", "Date"]]

city_stats.describe()
```
```
# Indices of cities that have humidity over 100%.
high_humidity = city_data.loc[city_data["Humidity"]> 100]
high_humidity
```

### Plotting the Data
* Latitude vs. Temperature Plot
* Latitude vs. Humidity Plot
* Latitude vs. Cloudiness Plot
* Latitude vs. Wind Speed Plot

### Linear Regression
* Northern Hemisphere - Max Temp (C) vs. Latitude Linear Regression
* Southern Hemisphere - Max Temp (C) vs. Latitude Linear Regression
* Northern Hemisphere - Humidity (%) vs. Latitude Linear Regression
* Southern Hemisphere - Humidity (%) vs. Latitude Linear Regression
* Northern Hemisphere - Cloudiness (%) vs. Latitude Linear Regression
* Southern Hemisphere - Cloudiness (%) vs. Latitude Linear Regression
* Northern Hemisphere - Wind Speed (m/s) vs. Latitude Linear Regression
* Southern Hemisphere - Wind Speed (m/s) vs. Latitude Linear Regression

## VacationPy
```
# Dependencies and Setup
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import requests
import gmaps
import os

# Import API key
from config import g_key


# Load in cities csv and store as a dataframe
cities_data = pd.read_csv("output_data/cities.csv")
cities_data
```
### Humidity Heatmap
```
# Configure gmaps
gmaps.configure(api_key=g_key)

# Store latitude and longitude in locations and convert humidity to float
locations = cities_data[["Lat", "Lng"]]
humidity = cities_data["Humidity"].astype(float)

# Plot Heatmap
fig = gmaps.figure()

# Create heat layer
heat_layer = gmaps.heatmap_layer(locations, weights=humidity, 
                                 dissipating=False, max_intensity=100,
                                 point_radius=1)

# Add layer
fig.add_layer(heat_layer)

# Display figure
fig
```

### Hotel Map
```
# Filter cities by ideal weather conditions
ideal_cities = cities_data.loc[((cities_data["Max Temp"] >= 26) & (cities_data["Wind Speed"] < 4.47)
                                & (cities_data["Humidity"] <= 50)) ]
# Drop null values
ideal_cities.dropna()
```
```
# Store filtered dataframe into new variable 
hotel_df = ideal_cities

# Add hotel name column
hotel_df["Hotel Name"] = ""

# Params dictionary to update each iteration
params = {
    "radius": 5000,
    "keyword": "hotel",
    "key": g_key
}

# Use cities' latitudes and longitudes to identify hotels
for index, row in hotel_df.iterrows():
    
    # Get lat, lng from df
    lat = row["Lat"]
    lng = row["Lng"]

    # Change location each iteration while leaving original params in place
    params["location"] = f"{lat},{lng}"

    # Use the search term: "hotel" and lat/lng
    base_url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"

    # Make request
    hotel_name = requests.get(base_url, params=params).json()

    # Pass any missing data with exception handling
    try:
        hotel_df.loc[index, "Hotel Name"] = hotel_name["results"][0]["name"]
    except (KeyError, IndexError):
        pass
```
```
# Using the template add the hotel marks to the heatmap
info_box_template = """
<dl>
<dt>Name</dt><dd>{Hotel Name}</dd>
<dt>City</dt><dd>{City}</dd>
<dt>Country</dt><dd>{Country}</dd>
</dl>
"""
# Store the DataFrame Row
hotel_info = [info_box_template.format(**row) for index, row in hotel_df.iterrows()]
locations = hotel_df[["Lat", "Lng"]]

# Add marker layer ontop of heatmap
markers = gmaps.marker_layer(locations, info_box_content=hotel_info)
# Add the layer to the map
fig.add_layer(markers)
fig

# Display figure
fig
```
