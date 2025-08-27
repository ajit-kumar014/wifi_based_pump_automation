# Wifi based Water pump automation
Turn your ordinary water pump into a smart, Wi-Fi powered system! Using ESP8266, an WaterProof ultrasonic sensor, and SSR-40A, this project automates tank filling, prevents overflow, saves water, and lets you monitor and control everything in real time via Home Assistant.


## Things Required.
1.  ESP8266 (NodeMCU/Lonin) 2 in Quantity
2.  Water-Proof Ultrasonic Sensor
3.  Solid State Relay(SSR-40DA)
4.  Battery 5V


## Setup Required before dealing with Hardware
1.  Docker installed on the system
    If not installed you can checkout [Docker installation guide](https://docs.docker.com/engine/install/)
2.  [Esphome](https://esphome.io) installed on docker:
    ```
    services:
       esphome:
        image: ghcr.io/imagegenius/esphome:latest
        container_name: esphome
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Etc/UTC
          - ESPHOME_DASHBOARD_USE_PING=false #optional
        volumes:
          - path_to_appdata:/config
        ports:
          - 6052:6052
        restart: unless-stopped
    ```
    * Just update the path_to_appdata field
    * Copy this to file named docker-compose.yml then save it.
    * Run `docker compose up -d` inside terminal (in the same folder as the compose file).
    * Then go to you browser the hit [http://localhost:6052](http://localhost:6052) for another device get the IP address of the system          running docker (IP_ADD:6052)
3.  (Optional) [HomeAssitant](https://www.home-assistant.io/) installation various ways but for this setup we will be using Container           based setup inside Docker.
     * Before setting up on Docker [Read this](https://www.home-assistant.io/installation/linux)
     Docker Compose file:
     ```
     services:
      homeassistant:
        container_name: homeassistant
        image: "ghcr.io/home-assistant/home-assistant:stable"
        volumes:
          - ./conf:/config
          - ./templates:/config/custom_templates
          - /etc/localtime:/etc/localtime:ro
        restart: unless-stopped
        healthcheck:
          test: ["CMD", "curl", "-f", "http://<your-domain>:8123"]
          interval: 1m
          timeout: 10s
          retries: 3
          start_period: 40s
        ports:
          - 8123:8123
        privileged: true
     ```
     * Keep in mind the volumes files must be present.
     * After that run `docker compose up -d` in terminal
     * Then go inside the browser and hit `localhost:8123` and go ahead and do the setup and explore.
## Steps for Hardware setup.
1.  Plug in your ESP module to system, then go to EspHome url.
2.  Click on the secrets option on top right corner, then paste this:
    ```
    # Your Wi-Fi SSID and password
    wifi_ssid: "YOUR_WIFI_NAME_HERE"
    wifi_password: "YOUR_WIFI_PASSWORD_HERE"
    # this is optional
    api_key: "YOUR_HOMEASSISTANT_API_FOR_ADDING_SUPPORT_FOR_HOMEASSITANT"
    ```
    then save.
3.  Go back then select " + New device ".
4.  Fill the name(This code below if for relay control, basically on/off control) that you want to give then select the appropriate board name then save.
5.  Open the .yaml that got created with the same name as provided.
6.  Make changes in this accordingly then paste it( keep in mind the GPIO pin connection for esp module and relay):
    ```
    esphome:
      name: relay
      friendly_name: relay
    
    esp8266:
      board: nodemcuv2
    
    # Enable logging
    logger:
      level: DEBUG
    
    # Enable Home Assistant API
    api:
      encryption:
        key: "YOUR_HOMEASSITANT_API_KEY"
    
    ota:
      - platform: esphome
        password: "dbddea81b8ffd0267fe9d781f42c625b"
    
    wifi:
      ssid: !secret wifi_ssid
      password: !secret wifi_password
      manual_ip:
        static_ip: 192.168.0.101
        gateway: 192.168.0.1
        subnet: 255.255.255.0
      
      # Enable fallback hotspot (captive portal) in case wifi connection fails
      ap:
        ssid: "Relay Fallback Hotspot"
        password: "gXBx50azyVMz"
    
    # Add web server for debugging and API access
    web_server:
      port: 80
    
    captive_portal:
    
    # Status LED
    status_led:
      pin:
        number: D4
        inverted: true
    
    # SSR Relay Switch
    switch:
      - platform: gpio
        pin: D2
        name: "SSR Relay"
        id: ssr_relay
        restore_mode: RESTORE_DEFAULT_OFF
        on_turn_on:
          then:
            - logger.log: "Pump turned ON by water level control"
            - globals.set:
                id: relay_state
                value: 'true'
        on_turn_off:
          then:
            - logger.log: "Pump turned OFF by water level control"
            - globals.set:
                id: relay_state
                value: 'false'
    
      # Manual override switch (for testing/emergency)
      - platform: template
        name: "Manual Override"
        id: manual_override
        restore_mode: RESTORE_DEFAULT_OFF
        turn_on_action:
          - logger.log: "Manual override enabled - pump control disabled"
          - globals.set:
              id: manual_mode
              value: 'true'
        turn_off_action:
          - logger.log: "Manual override disabled - automatic control resumed"
          - globals.set:
              id: manual_mode
              value: 'false'
    
      # Restart switch
      - platform: restart
        name: "Restart"
    
    # Global variables
    globals:
      - id: relay_state
        type: bool
        restore_value: true
        initial_value: 'false'
      - id: manual_mode
        type: bool
        restore_value: true
        initial_value: 'false'
    
    # Binary sensors
    binary_sensor:
      - platform: status
        name: "Status"
      
      # Template sensor to show if relay is controlled automatically
      - platform: template
        name: "Auto Control Active"
        id: auto_control_active
        lambda: |-
          return !id(manual_mode);
    
    # Text sensors
    text_sensor:
      - platform: template
        name: "Relay Status"
        id: relay_status
        lambda: |-
          if (id(manual_mode)) {
            return {"Manual Mode"};
          } else if (id(relay_state)) {
            return {"Auto - Pump ON"};
          } else {
            return {"Auto - Pump OFF"};
          }
        update_interval: 5s
    
      # Device info
      - platform: version
        name: "ESPHome Version"
        hide_timestamp: true
    
      # WiFi Info
      - platform: wifi_info
        ip_address:
          name: "IP Address"
        ssid:
          name: "Connected SSID"
    
    # Sensors
    sensor:
      # WiFi Signal
      - platform: wifi_signal
        name: "WiFi Signal"
        update_interval: 60s
    
      # Uptime
      - platform: uptime
        name: "Uptime"
        filters:
          - lambda: return x / 3600;  # Convert to hours
        unit_of_measurement: "h"
        accuracy_decimals: 1
    
    # Interval for status logging
    interval:
      - interval: 60s
        then:
          - logger.log: 
              format: "Relay Status: %s - Manual Mode: %s"
              args: [ 'id(relay_state) ? "ON" : "OFF"', 'id(manual_mode) ? "ENABLED" : "DISABLED"' ]
    ```
    * Then Click install, then select 'Plug into this computer' option then wait for it to get installed.
    * After that Unplug then go with the wiring process.
    * GPIO Pins for ESP Module is:
    ```
     ┌───────────────┐
     │   NodeMCU     │
     │ (ESP8266)     │
     │               │
     │  D4 (GPIO2) ──> Onboard LED (Status)  
     │               │
     │  D2 (GPIO4) ──> SSR Relay (Control Pin)  
     │               │
     │  GND ─────────> SSR Relay GND  
     │  3V3/5V ──────> SSR Relay VCC (depends on relay type)  
     │               │
     └───────────────┘ 
    
     ┌───────────────┐
     │   SSR Relay   │
     │               │
     │ Input: VCC, GND, IN (from D2)  
     │ Output: AC Live (Pump control)  
     │               │
     └───────────────┘
    ```
    * for more details [Checkout this video on Youtube](https://youtu.be/DZrOOhRCtZM) but instead of Arduino we are using the espboard.
    * Try to test is out using LEDs or any mobile charger by going to the webpage present at manual_ip(if not showing then check out your network subnet the change the .yaml file          then retry again).
7.  Then make the connection with the motor (I would recommend to create a separate Socket board for that purpose with an MCB as a safety measure)
8.  Please take Help of Electrician for this DO NOT TRY THIS ON YOUR OWN!!!.
9.  Now setting up the WaterProof sensor and another ESP module for sending the tank data to the relay ESP module.
10.  Follow the same steps for ESP module configuration.
11.  .yaml file :
        ```
        esphome:
      name: nodemcu-water-level
      friendly_name: Water Level Monitor
    
        esp8266:
          board: nodemcuv2
        
        logger:
          level: DEBUG
        
        wifi:
          ssid: !secret wifi_ssid
          password: !secret wifi_password
          manual_ip:
            static_ip: 192.168.0.100
            gateway: 192.168.0.1
            subnet: 255.255.255.0
          ap:
            ssid: "Water-Level-Fallback"
            password: "your-fallback-password"
        
        ota:
          - platform: esphome
            password: "4980cd8a80a1188c42856fd3c3d1ac45"
        
        api:
          encryption:
            key: "HOME_ASSITANT_API_KEY"
        
        # Web Server
        web_server:
          port: 80
        
        # HTTP Request component for communicating with relay ESP
        http_request:
          useragent: esphome/device
          timeout: 10s
          verify_ssl: false
        
        # Status LED
        status_led:
          pin:
            number: D4
            inverted: true
        
        # Sensors
        sensor:
          # Ultrasonic Distance Sensor (Raw Distance)
          - platform: ultrasonic
            trigger_pin:
              number: D5
              inverted: true
            echo_pin: D6
            name: "Water Distance Raw"
            id: water_distance_sensor
            update_interval: 2s
            timeout: 4m
            pulse_time: 10us
            internal: true  # Hide from HA as we'll use calculated values
            filters:
              - filter_out: nan
              - median:
                  window_size: 5
                  send_every: 3
              - lambda: |-
                  if (x > 3.0) {
                    return {};  // Filter out readings > 3m (likely errors)
                  }
                  return x;
        
          # Water Level (calculated from distance)
          - platform: template
            name: "Water Level"
            id: water_level
            unit_of_measurement: "m"
            device_class: distance
            state_class: measurement
            accuracy_decimals: 2
            icon: "mdi:water-well"
            lambda: |-
              float tank_height = 1.5;           // Total tank height in meters
              float sensor_offset = 0.3;         // Sensor is 30cm below tank top
              float max_water_height = 1.2;      // Maximum water height (1.5m - 0.3m)
              float sensor_distance = id(water_distance_sensor).state;
              
              if (isnan(sensor_distance)) {
                return {};
              }
              
              // Calculate water level from sensor position
              float water_level = max_water_height - sensor_distance;
              
              // Ensure water level is within valid range
              if (water_level < 0) water_level = 0;
              if (water_level > max_water_height) water_level = max_water_height;
              
              return water_level;
            update_interval: 2s
        
          # Water Level Percentage
          - platform: template
            name: "Water Level Percentage"
            id: water_level_percentage
            unit_of_measurement: "%"
            state_class: measurement
            accuracy_decimals: 1
            icon: "mdi:water-percent"
            lambda: |-
              float tank_height = 1.5;           // Total tank height in meters
              float sensor_offset = 0.3;         // Sensor is 30cm below tank top
              float max_water_height = 1.2;      // Maximum water height (1.5m - 0.3m)
              float sensor_distance = id(water_distance_sensor).state;
              
              if (isnan(sensor_distance)) {
                return {};
              }
              
              // Calculate water level from sensor position
              float water_level = max_water_height - sensor_distance;
              
              // Ensure water level is within valid range
              if (water_level < 0) water_level = 0;
              if (water_level > max_water_height) water_level = max_water_height;
              
              // Calculate percentage based on maximum possible water height
              return (water_level / max_water_height) * 100;
            update_interval: 2s
            on_value:
              then:
                - script.execute: control_pump
        
          # Water Volume (for circular tank with 1m diameter)
          - platform: template
            name: "Water Volume"
            id: water_volume
            unit_of_measurement: "L"
            state_class: measurement
            accuracy_decimals: 0
            icon: "mdi:water"
            lambda: |-
              float tank_height = 1.5;           // Total tank height in meters
              float sensor_offset = 0.3;         // Sensor is 30cm below tank top
              float max_water_height = 1.2;      // Maximum water height (1.5m - 0.3m)
              float tank_radius = 0.5;           // Tank radius = 0.5m (diameter 1m)
              float pi = 3.14159;
              float sensor_distance = id(water_distance_sensor).state;
              
              if (isnan(sensor_distance)) {
                return {};
              }
              
              // Calculate water level from sensor position
              float water_level = max_water_height - sensor_distance;
              
              // Ensure water level is within valid range
              if (water_level < 0) water_level = 0;
              if (water_level > max_water_height) water_level = max_water_height;
              
              // Volume = π × r² × h (in cubic meters, then convert to liters)
              float volume_m3 = pi * tank_radius * tank_radius * water_level;
              return volume_m3 * 1000;  // Convert to liters
            update_interval: 2s
        
          # WiFi Signal Strength
          - platform: wifi_signal
            name: "WiFi Signal"
            update_interval: 60s
        
          # Uptime
          - platform: uptime
            name: "Uptime"
            filters:
              - lambda: return x / 3600;  # Convert to hours
            unit_of_measurement: "h"
            accuracy_decimals: 1
        
        # Binary Sensors
        binary_sensor:
          # Water Level Alerts
          - platform: template
            name: "Water Level Low"
            id: water_low
            device_class: problem
            lambda: |-
              return id(water_level_percentage).state < 35;  // Alert when < 35%
            
          - platform: template
            name: "Water Level Critical"
            id: water_critical
            device_class: problem
            lambda: |-
              return id(water_level_percentage).state < 20;  // Critical when < 20%
            
          - platform: template
            name: "Water Level High"
            id: water_high
            lambda: |-
              return id(water_level_percentage).state > 90;  // High when > 90%
        
          # Connection Status
          - platform: status
            name: "Status"
        
        # Text Sensors
        text_sensor:
          # Water Level Status
          - platform: template
            name: "Water Level Status"
            id: water_level_status
            icon: "mdi:water-alert"
            lambda: |-
              float percentage = id(water_level_percentage).state;
              if (isnan(percentage)) {
                return {"Unknown"};
              } else if (percentage < 20) {
                return {"Critical"};
              } else if (percentage < 35) {
                return {"Low"};
              } else if (percentage < 80) {
                return {"Normal"};
              } else {
                return {"High"};
              }
            update_interval: 5s
        
          # Device Info
          - platform: version
            name: "ESPHome Version"
            hide_timestamp: true
        
          # WiFi Info
          - platform: wifi_info
            ip_address:
              name: "IP Address"
            ssid:
              name: "Connected SSID"
        
        # Switches (for testing/debugging)
        switch:
          - platform: restart
            name: "Restart"
        
        # Global variables to track pump state
        globals:
          - id: pump_state
            type: bool
            restore_value: true
            initial_value: 'false'
        
        # Script to control pump based on water level
        script:
          - id: control_pump
            then:
              - lambda: |-
                  float water_percentage = id(water_level_percentage).state;
                  bool current_pump_state = id(pump_state);
                  
                  if (isnan(water_percentage)) {
                    ESP_LOGW("pump_control", "Water level reading is NaN, skipping control");
                    return;
                  }
                  
                  ESP_LOGI("pump_control", "Water level: %.1f%%, Current pump state: %s", 
                           water_percentage, current_pump_state ? "ON" : "OFF");
              - if:
                  condition:
                    lambda: |-
                      float water_percentage = id(water_level_percentage).state;
                      bool current_pump_state = id(pump_state);
                      return (water_percentage <= 30.0 && !current_pump_state);
                  then:
                    - lambda: |-
                        ESP_LOGI("pump_control", "Water level low (%.1f%%), turning pump ON", id(water_level_percentage).state);
                        id(pump_state) = true;
                    - http_request.post:
                        url: "http://192.168.0.101/switch/ssr_relay/turn_on"
                        request_headers:
                          Content-Type: application/json
                        on_response:
                          then:
                            - lambda: |-
                                ESP_LOGI("pump_control", "HTTP Response: %d", response->status_code);
                                if (response->status_code == 200) {
                                  ESP_LOGI("pump_control", "Successfully turned pump ON");
                                } else {
                                  ESP_LOGI("pump_control", "Failed to turn pump ON");
                                }
              - if:
                  condition:
                    lambda: |-
                      float water_percentage = id(water_level_percentage).state;
                      bool current_pump_state = id(pump_state);
                      return (water_percentage >= 80.0 && current_pump_state);
                  then:
                    - lambda: |-
                        ESP_LOGI("pump_control", "Water level high (%.1f%%), turning pump OFF", id(water_level_percentage).state);
                        id(pump_state) = false;
                    - http_request.post:
                        url: "http://192.168.0.101/switch/ssr_relay/turn_off"
                        request_headers:
                          Content-Type: application/json
                        on_response:
                          then:
                            - lambda: |-
                                ESP_LOGI("pump_control", "HTTP Response: %d", response->status_code);
                                if (response->status_code == 200) {
                                  ESP_LOGI("pump_control", "Successfully turned pump OFF");
                                } else {
                                  ESP_LOGI("pump_control", "Failed to turn pump OFF");
                                }
        
        # Intervals for periodic actions
        interval:
          - interval: 30s
            then:
              - logger.log: 
                  format: "Water Level: %.2fm (%.1f%%) - Status: %s - Pump: %s"
                  args: [ 'id(water_level).state', 'id(water_level_percentage).state', 'id(water_level_status).state.c_str()', 'id(pump_state) ? "ON" : "OFF"' ] 
        ```
11.     Save it then Install using the Plug into this Computer option.
12.     wait for installation.
13.     The go to the IP_address present in the manual_ip section in .yaml file
14. 
        
