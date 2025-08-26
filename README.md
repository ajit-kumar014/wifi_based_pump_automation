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
    `services:
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
        restart: unless-stopped`
    * Just update the path_to_appdata field
3.  (Optional) [HomeAssitant](https://www.home-assistant.io/) installation various ways but for this setup we will be using Container based setup inside Docker
