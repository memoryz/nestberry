# Recording Nest Video with Raspberry Pi

## Why?

Google Nest does not record motion-triggered video unless you subscribe to Nest Aware service. The [service cost](https://store.google.com/us/product/nest_aware) is $60/year for 30 days event video history, and $120/year for 60 days history.

Meanwhile, Google Nest provides APIs (Device Access Console) for camera or doorbell events, and we can setup our own video recording service on a Raspberry Pi, with a USB thumb drive as the external storage for recordings.

## Total cost

| Component                                        | Cost      |
|--------------------------------------------------|-----------|
| Google DAC registration fee                      | $5        |
| CanaKit Raspberry Pi 4 kit with fan (4GB RAM)    | $82.99    |
| Micro SD Card 32GB or more (Application Class 2) | $10 - $20 |
| Any USB Thumb Drive 4GB or more                  | $10 - $20 |

Additionally you will need a micro SD card reader if you don't already have one.

Amazon links are posted [below](#resources).

## Software Requirements

1. [Home Assistant OS image](https://www.home-assistant.io/hassio/installation/) for Raspberry Pi 4 Model B

    > Either [32-bit](https://github.com/home-assistant/operating-system/releases/download/5.9/hassos_rpi4-5.9.img.xz) or [64-bit](https://github.com/home-assistant/operating-system/releases/download/5.9/hassos_rpi4-64-5.9.img.xz) should work.

2. [Raspberry Pi Manager](https://www.raspberrypi.org/software/) or [balenaEtcher](https://www.balena.io/etcher) for writing the OS image to the micro SD card.

## Installation

Follow the instruction [here](https://www.home-assistant.io/getting-started/#installation).

If you are using Raspberry Pi Manager, you can load the `hassos` image by clicking "Choose OS", then "Use custom".

## Configuration

1. Wireless Network (optional)

    If you installed the Home Assistant OS with ethernet cable, you might want to setup wireless network afterwards.

    [Here](https://community.home-assistant.io/t/guide-connecting-pi-with-home-assistant-os-to-wifi-or-other-networking-changes/98768) is the easiest method. If you run into a HTTP/502 error when launching the SSH, make sure you go to the addon configuration page and specify a password and change the SSH port from "Disabled" to 22.

    ![SSH Addon Config](/img/ssh_config.png)

2. Nest Integration

    To set up Nest integration, go to [this page](https://www.home-assistant.io/integrations/nest/), and follow these instructions:
    * [Device Access Registration](https://www.home-assistant.io/integrations/nest/#device-access-registration)
    * [Pub/Sub subscriber setup](https://www.home-assistant.io/integrations/nest/#pubsub-subscriber-setup)
    * [Configuration](https://www.home-assistant.io/integrations/nest/#configuration)
    * [Device Setup](https://www.home-assistant.io/integrations/nest/#device-setup)

    A few things to call out:
    1. When configuring the OAuth Client ID redirect URI (step 17), if you specify `http://homeassistant.local:8123/auth/external/callback`, you will see an error:

        ![Redirect URI error](/img/redirect_uri_error.png)

        To get around the error, simply change the domain in the redirect URI to a top-level domain (e.g. `homeassistant.local.com`), then add the domain to `/etc/hosts` or `C:\Windows\System32\drivers\etc\hosts` on your PC and map it to the IP address of the Raspberry Pi.

    2. The `configuration.yml` file is located under `/config/` directory. You will need SSH access. The instructions can be found in the **Wireless Network** section.

    3. We also need to enable `stream` integration for video recording.

   The `configuration.yml` file so far should look like this:

   ```yml
   # Configure a default setup of Home Assistant (frontend, api, etc)
   default_config:

   # Text to speech
   tts:
     - platform: google_translate

   group: !include groups.yaml
   automation: !include automations.yaml
   script: !include scripts.yaml
   scene: !include scenes.yaml

   nest:
     client_id: OAUTH_CLIENT_ID
     client_secret: OAUTH_CLIENT_SECRET
     # "Project ID" in the Device Access Console
     project_id: DAC_PROJECT_ID
     # Provide the full path exactly as shown under "Subscription name" in Google Cloud Console
     subscriber_id: SUBSCRIBER_ID

   stream:
   ```

3. External Storage

    We want to store the video recording on an external USB thumb drive. Mounting a USB drive to Home Assistant can be quite tricky, and there are a few ways to do it. The easiest way I found is [this](https://community.home-assistant.io/t/solved-mount-usb-drive-in-hassio-to-be-used-on-the-media-folder-with-udev-customization/258406). To summarize:

    1. Prepare the USB drive for configuration.
    From your desktop, format the USB drive with FAT32, and label it as `CONFIG` (case sensitive). Then create a folder called `udev`, and create a file called `80-mount-usb-to-media-by-label.rules` inside it. Copy the content from [this gist](https://gist.github.com/eklex/c5fac345de5be9d9bc420510617c86b5) to the file.

    2. Load the USB drive.
    Insert the USB drive into the Raspberry Pi, then go to HA Web UI, and go to `Supervisor`, `System`, and `Import from USB`.

    3. Prepare the USB drive for media.
    Move the USB drive to your desktop, wipe out the content, and rename the drive to anything but `CONFIG` (e.g. `RECORDING`). The `udev` rule will ignore any USB drive labeled as `CONFIG`.

    4. Plug in the USB drive to Raspberry Pi, and reboot HA from the HA Web UI (`Configuration`, `Server Controls`, `Server management`, `RESTART`).

    5. Verify the drive is mounted.
    Go to HA Web UI, open SSH, and make sure the USB drive is mounted under the `/media` directory.

        ```bash
        ~$ ls /media/
        RECORDING
        ```

    6. Add the mounted folder to HA allowed list.
    Modify `/config/configuration.yml` file and add the following section:

        ```yml
        homeassistant:
          allowlist_external_dirs:
            - /media/RECORDING
        ```

4. Preload the stream

    Go to HA Web UI. In the `Overview` page, the Nest camera feeds should have been added to the page already. If not, you can edit the dashboard, and `Add Card`, `BY ENTITY`, and select the Nest cameras.

    Click each camera card, and a video player should start to stream the camera feed. On the top right corner, there is a checkbox "Preload stream". Make sure it is selected for every camera feed.

    ![Preload stream](img/preload_stream.png)

5. Automation
    Go to the `Configuration`, `Automations` page, and start adding automation. You can either `edit with UI` or `edit as YAML`. Here is an example:

    ```yml
    alias: Backyard Motion Detected
    description: ''
    trigger:
      - platform: device
        device_id: 290b2e43e52fdf90f48053e145ea8ce3
        domain: nest
        type: camera_person
      - platform: device
        device_id: 290b2e43e52fdf90f48053e145ea8ce3
        domain: nest
        type: camera_motion
    condition: []
    action:
      - service: camera.record
        data:
          filename: '/media/RECORDING/{{ now().strftime("%Y/%m/%d/backyard_%H%M%S") }}.mp4'
          duration: 60
          lookback: 5
        entity_id: camera.backyard
    mode: single
    ```

    > For the `device_id` field, you can go to the `Configuration`, `Devices` page, select the camera, select the camera, and the `device_id` is the GUID in the URL.

    When the Nest camera receives a 'Person detected' or 'Motion detected' event, HA will record around 60 seconds of the video feed to the `/media/RECORDING/%Y/%m/%d/` folder, backed by the USB drive. You can also view the recording in the `Media Browser` tab directly in HA Web UI.

## Resources

1. [CanaKit Raspberry Pi 4 kit with fan (4GB RAM)](https://www.amazon.com/gp/product/B07VYC6S56/ref=ppx_yo_dt_b_asin_title_o00_s01)

2. [Micro SD Card 64GB](https://www.amazon.com/gp/product/B07FCMBLV6/ref=ppx_yo_dt_b_asin_title_o00_s02)

3. [Micro SD Card Reader](https://www.amazon.com/gp/product/B0046TJG1U/ref=ppx_yo_dt_b_search_asin_title)
