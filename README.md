# WeatherStation2HomeAssisant
How to load Weatherstation data into Home Asisstant




Howto parse a weatherstation on Weathercloud website.

- install Alpine linux
https://alpinelinux.org/downloads/

- Update and install docker on Alpine linux
apk update
apk add docker
addgroup $user docker  #add yourself to the users of docker
rc-update add docker default
service docker start
apk add docker-cli-compose
docker run -d -p 4444:4444 -p 7900:7900 --shm-size="2g" selenium/standalone-chrome:latest #run docker for selenium

- Install python
apk add python3
apk add py3-pip
apk add py3-paho-mqtt
pip3 install -U selenium –break-system-packages

- Install the code
/home/henk # cat Weatherstation.py 
#   Code to fetch the weather details from the weatherCloud website
#
#   Created by Henk Uyttenhove - 24/08/24

# Define the libraries
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time
import sys
import random

# mqtt prepare connection
import paho.mqtt.client as paho

def on_connect(client, userdata, flags, rc):
    print('CONNACK received with code %d.' % (rc))

broker = '172.16.0.9'
port = 1883
topic = "weatherstation/"
username = 'telescoop'
password = 'gpz1100'

client = paho.Client()
client.on_connect = on_connect
client.username_pw_set(username,password)
client.connect(broker, port)



#client.connect("14b5793c334743769b3e9fb1e4008401.s2.eu.hivemq.cloud", 8883)
#client.publish("encyclopedia/temperature", payload="hot", qos=1)

#  Put here the link to the required weatherstation but look on the map
#  of WeatherCloud and klik on the bullet to get the link
#  Replace the URL below with your URL

# URL =  "https://app.weathercloud.net/map#YOUR_STATION"
URL = "https://app.weathercloud.net/map#8283373349"

# Create a browser instance to emulate and fetch data
options = webdriver.ChromeOptions()
driver = webdriver.Remote(command_executor="http://localhost:4444", options=options)

#driver = webdriver.Chrome()
# Open the web page
driver.get(URL)
# Perform interactions (clicks, scrolls, etc.)
driver.maximize_window()
WebDriverWait(driver, 60).until(EC.element_to_be_clickable((By.XPATH, "//button[contains(., 'Consent')]"))).click()
time.sleep(2)
WebDriverWait(driver, 60).until(EC.element_to_be_clickable((By.CLASS_NAME, "side-box-variables")))
# Extract data
data = driver.find_element(By.CLASS_NAME, 'side-box-variables')
if not data:
    print("Fetch failed")
    sys.exit()
WeatherString = str(data.text)
driver.quit()

# Split the textblox in lines
splitWeatherString = WeatherString.split('\n')

# Get the values from the string 
position = splitWeatherString[1].find("m/s")
Windspeed = float(splitWeatherString[1][0:position])
WindDirection = splitWeatherString[1][-1:]

# Get the temperature
position = splitWeatherString[3].find("°")
Temperature = float(splitWeatherString[3][0:position])

# Get the humidity
position = splitWeatherString[5].find("*")
Humidity = float(splitWeatherString[5][0:position])

# Get the Air Pressure
position = splitWeatherString[7].find("hPa")
AirPressure = float(splitWeatherString[7][0:position])

# Get the Rain and RainSpeed
position = splitWeatherString[9].find("mm")
Rain = float(splitWeatherString[9][0:position])
position = splitWeatherString[11].find("mm/h")
RainSpeed = float(splitWeatherString[11][0:position])

# Get the SolarIntensity
position = splitWeatherString[13].find("W/m")
SolarIntensity = float(splitWeatherString[13][0:position])

# Get the UV
position = splitWeatherString[15].find(" ")
UV = float(splitWeatherString[15][0:position])
UVIntensity = splitWeatherString[15][position+1:]

print("Wind:", Windspeed, "meter per sec vanuit ", WindDirection)
print("Temperature:", Temperature, "graden")
print("Humidity:", Humidity)
print("Luchtdruk:", AirPressure)
print("Regen:", Rain, "mm aan snelheid ", RainSpeed, "mm/h")
print("Zonnestraling:", SolarIntensity, " W/sqm")
print("UV index", UV, " met intensiteit ", UVIntensity)

#Publish all to Home Assistant
client.publish(topic+"WindSpeed", payload=Windspeed, qos=1)
client.publish(topic+"WindDirection", payload=WindDirection, qos=1)
client.publish(topic+"Temperature", payload=Temperature, qos=1)
client.publish(topic+"Humidity", payload=Humidity, qos=1)
client.publish(topic+"Airpressure", payload=AirPressure, qos=1)
client.publish(topic+"Rain", payload=Rain, qos=1)
client.publish(topic+"RainSpeed", payload=RainSpeed, qos=1)
client.publish(topic+"SolarIntensity", payload=SolarIntensity, qos=1)
client.publish(topic+"UVIntensity", payload=UVIntensity, qos=1)

/home/henk # crontab -l
# do daily/weekly/monthly maintenance
# min	hour	day	month	weekday	command
*/5	*	*	*	*	/usr/bin/python3 /home/henk/Weatherstation.py

-activate in HomeAssistant

MQTT.yaml:

sensor:
- name: "NINA Sequence Status"
state_topic: "Astro/NINA/sequencestatus"
unique_id: jfgikmfuhgnghn

- name: "NINA Target"
state_topic: "Astro/NINA/target/name"
unique_id: y7ubrdyur6ub

- name: "NINA Camera Temp"
state_topic: "Astro/NINA/equipment/camtemp"
unit_of_measurement: "°C"
unique_id: w5csryju8

- name: "NINA Frame"
state_topic: "Astro/NINA/frame"
unique_id: w5csryju8tseyhe6

- name: "Starcount"
state_topic: "SKYSTATE"

- name: "WindSpeed"
state_topic: "weatherstation/WindSpeed"
unit_of_measurement: "m/sec"

- name: "WindDirection"
state_topic: "weatherstation/WindDirection"

- name: "Temperature"
state_topic: "weatherstation/Temperature"
unit_of_measurement: "°C"

- name: "Humidity"
state_topic: "weatherstation/Humidity"
unit_of_measurement: "%"

- name: "Airpressure"
state_topic: "weatherstation/Airpressure"
unit_of_measurement: "hPa"

- name: "Rain"
state_topic: "weatherstation/Rain"
unit_of_measurement: "mm"

- name: "RainSpeed"
state_topic: "weatherstation/RainSpeed"
unit_of_measurement: "mm/h"

- name: "UVIntensity"
state_topic: "weatherstation/UVIntensity"

configuration.yaml:

mqtt: !include mqtt.yaml
