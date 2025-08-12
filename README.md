# Home Assistant Voice Pipeline Guide with Wyoming Satellite
Getting into the Home Assistant ecosystem can be daunting at first. That's why I've written this guide to share my experiences

## Hardware:
### Wyoming Satellite
- [Raspberry Pi 3B+](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/)
- [DUNGZDUZ CGS-M3 USB Microphone](https://a.co/d/2fo1YoS)
### Home Assistant Server
- [Sonoff Zigbee USB Dongle](https://a.co/d/bm4GIn3)
- CPU: AMD Ryzen 5 5600X
- GPU: Nvidia RTX 3090
- RAM: 64GB DDR4
### Lights
- [ThirdReality Zigbee Smart Color Bulb](https://a.co/d/2n18fpk)

You do not have to use the exact same hardware for everything but there are a few things to take note of. You need a Zigbee dongle that can connect to Zigbee2MQTT and lights that are compatible as well. Also lights that can act as routers are great because they boost the Zigbee signal (the ThirdReality Zigbee lights are routers).

Honestly, these ThirdReality Zigbee lights are so cool. $30 USD for 4 bulbs that can, change brightness, color temperature,  RGB color (though the colors leave something to be desired; pictures below), act as Zigbee routers and easily connect to Zigbee2MQTT.

<details>
  <summary>ThirdReality RGB Colors</summary>

Both Red and blue look fine but green is okay with yellow having a lot of green mixed in (yellow actually looks better in the picture than real world). I didn't buy these lights for the colors but it's something to be mindful of.
![ThirdReality Light Blue](https://github.com/user-attachments/assets/4d794038-fb15-41ab-8a82-466bc582768c)

![ThirdReality Light Red](https://github.com/user-attachments/assets/ff511949-b1b0-4b8a-ac92-8d466e2e3e2a)

![ThirdReality Light Green](https://github.com/user-attachments/assets/f2afa59d-b255-4f8c-9856-2207371e4e6c)

![ThirdReality Light Yellow](https://github.com/user-attachments/assets/06fbfbc3-cf5f-4454-9c06-20f46acf330d)

</details>

## Prerequisites
Most guides will reference [Home Assistant OS](https://www.home-assistant.io/installation#about-installation-methods) and its addons but I already have a [server](https://www.truenas.com/truenas-community-edition/) running its own services so instead we'll be using Home Assistant's Docker container. The server you intend to run Home Assistant on should be running Linux under the hood, have [Docker Compose](https://docs.docker.com/desktop/setup/install/linux/) installed and at minimum a mid-range CPU with 12GB system RAM. While not strictly required is it recommended that you also have a GPU with at lease 6GB VRAM or the time to execute your voice commands will be slow.

## Speach to Text and Text to Speach
Copy the following into a filed named `docker-compose-speaches.yaml` and run `docker-compose -f docker-compose-speaches.yaml up`. This will download the [speaches](https://github.com/speaches-ai/speaches) and [wyoming-openai](https://github.com/roryeckel/wyoming_openai) Docker images. The init-speaches container will automatically download the [speach-to-text](https://huggingface.co/rtlingo/mobiuslabsgmbh-faster-whisper-large-v3-turbo) and [text-to-speach](https://huggingface.co/speaches-ai/Kokoro-82M-v1.0-ONNX) models.
<details>
  <summary>docker-compose-speaches.yaml</summary>
  
  ```
  networks:
    speaches_network:
      driver: bridge
      name: speaches_network
    wyoming_openai_network:
      driver: bridge
      name: wyoming_openai_network
  
  volumes:
    huggingface-hub:
  
  services:
    init-speaches: # Sidecar container to automatically download the STT and TTS models at startup.
      command: |
        /bin/bash -c '
          uv tool run speaches-cli model download speaches-ai/Kokoro-82M-v1.0-ONNX
          uv tool run speaches-cli model download rtlingo/mobiuslabsgmbh-faster-whisper-large-v3-turbo
        '
      container_name: init-speaches
      depends_on:
        speaches:
          condition: service_healthy
      deploy:
        resources:
          reservations:
            devices:
              - capabilities:
                  - gpu
                driver: nvidia
      environment:
        - SPEACHES_BASE_URL=http://speaches:8000
      image: ghcr.io/speaches-ai/speaches:latest-cuda-12.4.1
      networks:
        - speaches_network
      restart: no
      volumes:
        - huggingface-hub:/home/ubuntu/.cache/huggingface/hub
  
    speaches: # Provides an OpenAIAPI endpoint for STT and TTS.
      container_name: speaches
      deploy:
        resources:
          reservations:
            devices:
              - capabilities:
                  - gpu
                driver: nvidia
      environment:
        - WHISPER__TTL=-1 # Ensures that the models always stay in memory for faster response times.
        - WHISPER__MODEL=rtlingo/mobiuslabsgmbh-faster-whisper-large-v3-turbo # My model on Huggingface. It's identical to https://huggingface.co/mobiuslabsgmbh/faster-whisper-large-v3-turbo but has the proper tags in the repository so Speaches AI can automatically download it.
        - WHISPER__compute_type=int8_float32
      healthcheck:
        interval: 30s
        retries: 3
        start_period: 40s
        test: ["CMD", "curl", "--fail", "http://0.0.0.0:8000/health"]
        timeout: 10s
      image: ghcr.io/speaches-ai/speaches:latest-cuda-12.4.1
      networks:
        - speaches_network
      ports:
        - '8000:8000'
      restart: unless-stopped
      volumes:
        - huggingface-hub:/home/ubuntu/.cache/huggingface/hub
  
    wyoming_openai: # Converts the OpenAI protocol to the Wyoming protocol so Home Assistant can use it.
      container_name: wyoming_openai
      depends_on:
        init-speaches:
          condition: service_healthy
      environment:
        STT_BACKEND: SPEACHES
        STT_MODELS: rtlingo/mobiuslabsgmbh-faster-whisper-large-v3-turbo
        STT_OPENAI_URL: http://speaches:8000/v1
        TTS_BACKEND: SPEACHES
        TTS_MODELS: speaches-ai/Kokoro-82M-v1.0-ONNX
        TTS_OPENAI_URL: http://speaches:8000/v1
        WYOMING_LANGUAGES: en
        WYOMING_LOG_LEVEL: INFO
        WYOMING_URI: tcp://0.0.0.0:10300
      image: ghcr.io/roryeckel/wyoming_openai:latest
      networks:
        - speaches_network
        - wyoming_openai_network
      ports:
        - '10300:10300'
      restart: unless-stopped
  ```
</details>

To check if it's working you can run the following to generate some audio.
```
curl "localhost:8000/v1/audio/speech" -s -H "Content-Type: application/json" \
  --output audio.wav \
  --data @- << EOF
{
  "input": "Hello World!",
  "model": "speaches-ai/Kokoro-82M-v1.0-ONNX",
  "voice": "af_aoede"
}
EOF
```
You should now have a file called `audio.wav`. Check if you can transcribe it.
```
curl -s "localhost:8000/v1/audio/transcriptions" -F "file=@audio.wav" -F "model=rtlingo/mobiuslabsgmbh-faster-whisper-large-v3-turbo"
```
You should get the following.
```
{"text":"Hello world!"}
```

## Large Language Model
Copy the following into a filed named `docker-compose-ollama.yaml` and run `docker-compose -f docker-compose-ollama.yaml up`. This will download the ollama Docker image. We will download the [large language model](https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF) later in the Home Assistant UI.
<details>
  <summary>docker-compose-speaches.yaml</summary>
  
  ```
  networks:
    ollama_network:
      driver: bridge
      name: ollama_network
  
  services:
    ollama:
      volumes:
        - ./ollama/ollama:/root/.ollama
      container_name: ollama
      pull_policy: always
      restart: unless-stopped
      image: docker.io/ollama/ollama:latest
      ports:
        - 11434:11434
      networks:
        - ollama_network
      deploy:
        resources:
          reservations:
            devices:
              - driver: nvidia
                count: all 
                capabilities: [gpu]
  ```
</details>

## Home Assistant and Zigbee2MQTT
Copy the following into a filed named `docker-compose-home-assistant.yaml`. This will download the Home Assistant, [Zigbee2MQTT](https://www.zigbee2mqtt.io/) and [Eclipse Mosquitto](https://mosquitto.org/) Docker images. Zigbee2MQTT will be used to pair and manage your Zigbee capable devices. Eclipse Mosquitto will be your MQTT broker allowing the devices to communicate with Home Assistant.

<details>
  <summary>docker-compose-home-assistant.yaml</summary>
  
  ```
  configs:
    mosquitto.conf:
      content: |
        # Changes in this file will be lost at the next restart
        # Use /mosquitto/config_includes directory for additional configuration
        listener 1883
  
        log_dest stdout
        allow_anonymous true
  
        persistence true
        persistence_location /mosquitto/data
        autosave_interval 1800
  
  networks:
    wyoming_openai_network:
      external: True
    ollama_network:
      external: True
    mqtt_network:
      driver: bridge
  
  services:
    homeassistant:
      container_name: homeassistant
      image: "ghcr.io/home-assistant/home-assistant:stable"
      restart: unless-stopped
      depends_on:
        - zigbee2mqtt
      networks:
        - mqtt_network
        - ollama_network
        - wyoming_openai_network
      ports:
        - '8123:8123'
      privileged: true
      volumes:
        - ./hass-config:/config
        - /etc/localtime:/etc/localtime:ro
        - /run/dbus:/run/dbus:ro
  
    zigbee2mqtt:
      container_name: zigbee2mqtt
      image: ghcr.io/koenkk/zigbee2mqtt
      restart: unless-stopped
      depends_on:
        - eclipse-mosquitto
      networks:
        - mqtt_network
      devices:
        - /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0:/dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0
      environment:
        ZIGBEE2MQTT_CONFIG_FRONTEND_ENABLED: 'true'
        ZIGBEE2MQTT_CONFIG_HOMEASSISTANT_ENABLED: 'true'
        ZIGBEE2MQTT_CONFIG_FRONTEND_HOST: 0.0.0.0
        ZIGBEE2MQTT_CONFIG_MQTT_BASE_TOPIC: zigbee2mqtt
        ZIGBEE2MQTT_CONFIG_MQTT_SERVER: mqtt://eclipse-mosquitto:1883
        ZIGBEE2MQTT_CONFIG_SERIAL_ADAPTER: ember
        ZIGBEE2MQTT_CONFIG_SERIAL_PORT: /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0
        ZIGBEE2MQTT_DATA: /app/data
      ports:
        - '8088:8080'
      volumes:
        - ./data:/app/data
        - /run/udev:/run/udev:ro
  
    eclipse-mosquitto:
      container_name: eclipse-mosquitto
      image: eclipse-mosquitto
      restart: unless-stopped
      networks:
        - mqtt_network
      configs:
        - source: mosquitto.conf
          target: /mosquitto/config/mosquitto.conf
      volumes:
        - ./data:/mosquitto/data
        - ./log:/mosquitto/log
  ```
</details>

Before you run `docker-compose -f docker-compose-home-assistant.yaml up` you will need to find your Zigbee dongle. To do so run `ls -l /dev/serial/by-id/` and find your Zigbee dongle. Mine is `usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0`.

Replace all `[your_zigbee_device]` with your device. So mine would look like.
```
devices:
  - /dev/serial/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0:/dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0
environment:
  ZIGBEE2MQTT_CONFIG_FRONTEND_ENABLED: 'true'
  ZIGBEE2MQTT_CONFIG_HOMEASSISTANT_ENABLED: 'true'
  ZIGBEE2MQTT_CONFIG_FRONTEND_HOST: 0.0.0.0
  ZIGBEE2MQTT_CONFIG_MQTT_BASE_TOPIC: zigbee2mqtt
  ZIGBEE2MQTT_CONFIG_MQTT_SERVER: mqtt://eclipse-mosquitto:1883
  ZIGBEE2MQTT_CONFIG_SERIAL_ADAPTER: ember
  ZIGBEE2MQTT_CONFIG_SERIAL_PORT: /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_a48b50fa3b53ef11b0212ce0174bec31-if00-port0
  ZIGBEE2MQTT_DATA: /app/data
```

If you have the same Zigbee dongle that I do, you may have to flash this [firmware](https://raw.githubusercontent.com/itead/Sonoff_Zigbee_Dongle_Firmware/refs/heads/master/Dongle-E/NCP_7.4.4/ncp-uart-sw_EZNet7.4.4_V1.0.0.gbl) to get it to work with the Ember adapter in Zigbee2MQTT. If you are not using the same Zigbee dongle you may skip this section but you might have to change this line `ZIGBEE2MQTT_CONFIG_SERIAL_ADAPTER: ember` in `docker-compose-home-assistant.yaml` for your dongle.

Now you can run `docker-compose -f docker-compose-home-assistant.yaml up`.

On your web browser naviage to `localhost:8080` Change any settings here if you need to and hit submit at the bottom.
<img width="3840" height="2160" alt="Zigbee2MQTT Onboarding" src="https://github.com/user-attachments/assets/3b985b6d-1e3a-4cab-951f-536563a826c4" />

Give it about a minute, close the tab and go back to `localhost:8080` (refreshing the page didn't seem to work for me). In the top middle click "Permit join (All)".
<img width="3840" height="2160" alt="Zigbee2MQTT Pairing Mode" src="https://github.com/user-attachments/assets/88e8a8a7-dd1d-4199-9bc9-1a20d90e5bad" />

Now plug in your light bulbs (preferably close to the Zigbee Dongle). If you haven't already plugged in your light bulbs they should flash warm white, cold white, red, green, blue and connect to Zigbee2MQTT.

If the lights don't flash and won't connect to Zigbee2MQTT you will have to put the light bulbs back into pairing mode by turning off and on the light 5 times in a row like so.

https://github.com/user-attachments/assets/19a81137-79b7-490d-b71f-41fe1b110120

You should now have a light connected to Zigbee2MQTT. I suggest naming it something you will remember it by like `ROOM-NAME_TYPE-OF-LIGHT` as it helps when you have a lot of lights. For now what I have is fine.

https://github.com/user-attachments/assets/4a8a90ae-a542-4ab3-88df-146168ea171b

Login to Home Assistant at `localhost:8123`, click `CREATE MY SMART HOME` and follow the prompts to create your account.

After you've created your account follow the steps outlined in the videos setup Home Assistant.

https://github.com/user-attachments/assets/bf4e6238-d86c-4d86-a964-eb78902aea7b

https://github.com/user-attachments/assets/c17767c5-1945-457a-9716-4df595e4c79b

https://github.com/user-attachments/assets/e5efab55-4978-4ff2-a0f1-fda2a13217d3

At this point the voice pipeline is setup. You can now use your phone or computer to talk commands into Home Assistant.

This is cool but it's inconvenient to have to open an app or website. That is why we'll also setup [Wyoming Satellite](https://github.com/rhasspy/wyoming-satellite). I won't go into detail on how to set it up as there is already an in depth guide [here](https://github.com/rhasspy/wyoming-satellite/blob/master/docs/tutorial_2mic.md).

After you've got your Wyoming Satellite working, take note of its IP address as we'll need that. Head back to Home Assistant. The IP address will be different from the video but the port should be the same if you haven't changed it. Ignore the prompts to "Set up voice assistant". They expect you to have Home Assistant OS installed for the fully local option and you won't be able to proceed. What we do works despite this.

https://github.com/user-attachments/assets/f5e5bd83-84b3-4ee0-9588-8aaf43597f40

Now you're ready to use your new LLM powered home assistant!

https://github.com/user-attachments/assets/f96b2bcf-a9b7-4992-9a58-3ef9b208c06e
