<h1 style="color:red;">seatconnect-homeassistant-guide</h1>

This guide will help you set up a workaround working for the discontinued seatconnect-homeassistant hacs plugin. in this example we'll get the position from the app into a homeassistant sensor.

<h2>What are the prerequisites?</h2>

1. a running iobroker instance. if you don't use iobroker yet, i would recommend you to start with a docker. [docker info](https://github.com/fl0rian089/seatconnect-homeassistant-guide/blob/9b08b13b76fb47f7863b8456a214fd59937a45ee/iobroker/dockerinfo.md) and [docker-compose.yaml](https://github.com/fl0rian089/seatconnect-homeassistant-guide/blob/9b08b13b76fb47f7863b8456a214fd59937a45ee/iobroker/docker-compose.yaml) here.
2. running homeassistant instance.
3. running Mosquitto broker on homeassistant.
4. If you want to use the googlemaps api, you need to create a google geocoding api key.

<h2>What to do first?</h2>

in ioBroker:
1. after you started iobroker, go to settings -> repositories and change the active repository to "beta", Click Save and Close.
2. click on adapter -> search for vw-connect click on it and install it.
3. click on adapter -> search for mqtt-client (the one with version 3.0.0) click on it and install it.
4. click on instances -> mqtt-client.0 -> settings (wrench).
   4.1. Host IP: ip-of-homeassistant instance
   4.2. Client-ID: would recommend to name it how you like.
   4.3. username / password (you can create a new user on homeassistant for better security, but you can just login with your admin account)
   4.4. if you want to use a prefix for subscribe and publish topics, choose one, I deleted everything.
5. Click Save and Close.
6. click on instances -> vw-connect.0 -> settings (wrench).
  6.1. connect app mail: <your-seat/cupra-account mail>
  6.2. connect app password: <your-seat/cupra-account password>
  6.3. Type: My SEAT (for SEAT & CUPRA cars)
  6.4. Update interval in minutes: choose what you like, i put 10 in.
  6.5. modify rest of settings as you want
  6.6. Click Save and Close.
7. check the logs for errors.
8. click on objects tab -> double-click vw-connect -> 0 -> <your-vin> -> position
  8.1. latitudeConv -> click the gear/settings symbol on the right -> Activate -> publish -> Activate & retain. -> Click Save and Close.
  8.2. longitudeConv -> click the gear/settings symbol on the right -> Activate -> publish -> Activate & retain. -> Click Save and Close.

<h2>Change to Homeassistant</h2>

1. get a file editor from the addons-store.
2. in file editor -> open configuration.yaml
3. in your mqtt area or if you haven't any at the bottom add the following:
```
mqtt:
  sensor:
    - name: "<VIN>_longitude_location"
      state_topic: "vw-connect/0/<vin>/position/longitudeConv"
    - name: "<VIN>_latitude_location"
      state_topic: "vw-connect/0/<vin>/position/latitudeConv"
```
4. to convert from lat/lon to address, you need a rest sensor in your template/sensor area:

   4.1 Google Geocoding API:
```     
  - platform: rest
    unique_id: "<VIN>_CarLocationAddress"
    name: "<VIN>_CarLocationAddress"
    resource_template: >
      {% set lat = states('sensor.<VIN>_latitude_location') | float(none) %}
      {% set lon = states('sensor.<VIN>_longitude_location') | float(none) %}
      {% if lat is not none and lon is not none %}
        https://maps.googleapis.com/maps/api/geocode/json?latlng={{ lat }},{{ lon }}&key=GOOGLE_API_KEY
      {% else %}
        ""
      {% endif %}
    method: GET
    headers:
      User-Agent: HomeAssistant
    value_template: >
      {%- if value_json is defined and value_json.results is defined and value_json.results | length > 0 -%}
        {%- set address = value_json.results[0].formatted_address -%}
        {%- set parts = address.split(", ") -%}
        {%- set street = parts[0].replace("straße", "str.").replace("Straße", "Str.") -%}                ### you might not need that. that's just to short the output a little bit for my sensor size ;)
        {%- set town = parts[1] if parts | length > 1 else "" -%}
        {{ street }}, {{ town }}
      {%- else -%}
         "Address not available"
      {%- endif %}
    scan_interval: 9999999999                  #### it's 9999999999 to prevent spamming the api. we'll add a trigger to our automations.yaml so the google api gets only contacted, if the position of the car changes.
```

  
   4.2 openstreetmap:
```     
- platform: rest
  unique_id: "<VIN>_CarLocationAddress"
  name: "<VIN>_CarLocationAddress"
  resource_template: >
    {% set lat = states('sensor.<VIN>_latitude_location') | float(none) %}
    {% set lon = states('sensor.<VIN>_longitude_location') | float(none) %}
    {% if lat is not none and lon is not none %}
      https://nominatim.openstreetmap.org/reverse?format=json&lat={{ lat }}&lon={{ lon }}
    {% else %}
      ""
    {% endif %}
  method: GET
  headers:
    User-Agent: HomeAssistant
  value_template: >
    {%- if value_json is defined and value_json.address is defined -%}
      {%- set road = value_json.address.road if value_json.address.road is defined else "" -%}
      {%- set house_number = value_json.address.house_number if value_json.address.house_number is defined else "" -%}
      {%- set postcode = value_json.address.postcode if value_json.address.postcode is defined else "" -%}
      {%- set town = value_json.address.town if value_json.address.town is defined else (
                    value_json.address.village if value_json.address.village is defined else (
                    value_json.address.city if value_json.address.city is defined else "")) -%}
      
      {%- set road_number = (road ~ ' ' ~ house_number).strip() -%}
      {%- set postcode_town = (postcode ~ ' ' ~ town).strip() -%}
      {%- set address_parts = [road_number, postcode_town] | reject('eq', '') | list -%}

      {%- if address_parts | length > 0 -%}
        {{ address_parts | join(', ') }}
      {%- else -%}
        "Address not available"
      {%- endif -%}
    {%- else -%}
      "Address not available"
    {%- endif %}

```
5. save file
6. open your automations.yaml in file editor:
```
- alias: "Update <VIN>_CarLocationAddress"
  trigger:
    - platform: state
      entity_id:
        - sensor.<VIN>_latitude_location
        - sensor.<VIN>_longitude_location
  action:
    - service: homeassistant.update_entity
      entity_id: sensor.<VIN>_CarLocationAddress
```
7. save file
8. dev-tools -> check yaml and restart homeassistant afterwards.
