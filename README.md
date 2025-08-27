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
4.  Fill the name that you want to give then select the appropriate board name then save.
5.  Open the .yaml that got created with the same name as provided.
6.  Make changes in this accordingly:
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

    
