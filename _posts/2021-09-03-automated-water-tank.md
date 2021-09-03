---
layout: post
title: "Automated Water Tank using Home Assistant and ESPHome"
---

# Water tank automation
Our house has a rainwater tank with a mains-powered pump. Annoyingly, the pump isn't smart enough to switch itself off when the level drops too low; instead, it'll keep trying to pump every 30 seconds or so. This gets quite annoying and presumably isn't good for the life of the pump. As the pump is also rather loud, it could be disturbing the neighbours if it runs during the night. So, I determined to "smart-ify" it!

## Parts
The parts used here are:

* **an ESP8266 or ESP32 board**.
* **a water level sensor:** I used the [DFRobot SEN0204](https://wiki.dfrobot.com/Non-contact_Liquid_Level_Sensor_XKC-Y25-T12V_SKU__SEN0204) which can be glued to the *outside* of the tank (magic!).
* **a Wi-Fi enabled smart plug:** I used [this one from Brilliant Smart](https://www.officeworks.com.au/shop/officeworks/p/brilliant-lighting-smart-wifi-plug-with-usb-bl2067605) because it's compatible with Home Assistant, has a USB socket to power the ESP board, it's cheap and readily available, and I had one spare.
* **a weatherproof enclosure:** [Mine is 171 x 121 x 80mm](https://www.jaycar.com.au/p/HB6129) and just barely big enough due to the USB socket being on the side of the smart switch.

## Wiring
The wiring is pretty straightforward and explained in the wiki entry linked above.

## ESPHome Configuration
Here is the YAML for ESPHome:

    {% highlight yml %}
    esphome:
    name: tank
    platform: ESP8266
    board: nodemcuv2

    # Enable logging
    logger:

    # Enable Home Assistant API
    api:

    ota:
    password: [redacted]

    wifi:
    ssid: [redacted]
    password: !secret wifi_password

    # Enable fallback hotspot (captive portal) in case wifi connection fails
    ap:
        ssid: "Tank Fallback Hotspot"
        password: [redacted]

    captive_portal:

    binary_sensor:
    - platform: gpio
        name: "Water Tank Level"
        id: water_tank_level
        pin:
        number: D5
    {% endhighlight %}

## Home Assistant Configuration

### Dashboard
With the ESPHome configuration done, I turned to the Home Assistant side of things. The sensor is presented as a simple binary sensor: `on` when the water is detected, `off` when it is not. The plug is a simple switch, either on or off.

To configure the dashboard elements, I used a horizontal stack. This is a neat card layout that allows you to keep related items together. There's probably a better way to do this--I'm far from a Home Assistant expert--but this works well enough. If you use the code below, make sure you set the entities correctly.

    {% highlight yml %}
    type: horizontal-stack
    title: Water Tank
    cards:
    - type: entity
      entity: switch.46043008c44f3387cebd
      name: Pump
      icon: hass:pump
    - type: entity
      entity: binary_sensor.water_tank_level
      name: Level
      icon: hass:water
    {% endhighlight %}

I also changed the entity type to `Sound` (go to `Configuration` --> `Customizations` --> select the entity --> `Device Class`). This means that the sensor data is presented as "Detected" instead of "On", which is more intuitive.

### Automation
Lastly, and the whole reason I did this project, the automation! I wanted the pump to switch off when **either**:

* the level drops below the sensor; or
* it's 10:30 PM.

And switch on if:

* the level is above the sensor; **and**
* it's 6:30 AM.

To do this, I created two automations. "Water Pump Off" has two Triggers: a device trigger for when the sensor stops detecting "sound", and a Time trigger for 10:30 PM. It then has one Action, to switch off the plug.

The second Automation, "Water Pump On", has a Time Trigger, a Condition (that the sensor is detecting "sound"), and an Action to switch on the plug.