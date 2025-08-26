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

     
