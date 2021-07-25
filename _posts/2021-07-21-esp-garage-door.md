---
layout: post
title: "ESP8266 Garage Door"
---

# Parts
The parts needed for this project are:

* an ESP8266 board (you could also use an ESP32 board). I used the NodeMCU ESP8266 12-E board, which I purchased from AliExpress.
* a 3.3v or 5v relay module.
* a surface mount magnetic reed switch. I used [this one from a local retailer](https://www.altronics.com.au/p/s5153-spdt-surface-mount-security-magnetic-reed-switch/).
* an ultrasonic distance sensor. I used the ubiquitous HC-SR04 because I had one on hand, but you'd be better off with the upgraded HC-SRF05 as it has a longer range.
* hookup wire, ideally a good quality one since you'll be using some pretty long runs.
* a power supply. The easiest option is a USB wall charger and micro-USB cable. Your old phone charger should work fine; I've used all mine on other projects, so I bought a multi-pack off Amazon.
* some sort of enclosure. You can use a simple project box or create a custom enclosure using a 3D printer or laser cutter.

# Software Prerequisites
Two open-source projects, which I'm sure you're aware of, make the software side of this project pretty straight-forward. These are [Home Assistant](https://home-assistant.io) and [ESPHome](https://esphome.io).

Getting these set up is beyond the scope of this post, but there are plenty of excellent resources for both projects. Each project also has excellent documentation.

# Wiring
You'll need to wire each of the components to the GPIO pins on the ESP board. Be aware that the pinouts do vary between boards (even boards using the ESP8266), but you can simply subsitute the values below with the appropriate pins on your board. Random Nerd Tutorials has an excellent guide to the ESP8266's GPIO, which you can find [here](https://randomnerdtutorials.com/esp8266-pinout-reference-gpios).

## Relay Module
The relay module will have an input side and an output side. The input side has three pins: positive, negative (ground) and data. Connect the positive a 3v pin, and connect the ground to a ground pin. Connect the data pin to pin D1 on the board.

On the output side, there are three screw terminals: NC (Normally Closed), NO (Normally Open) and a common pin. Since we're using the relay as a switch, it'll only be "on" when we're pressing the switch. So, we'll set up the relay as Normally Open, meaning that the circuit is normally **incomplete**. You'll wire the NO pin and the common pin into the garage door switch later.

## Reed Switch
Give your switch power from an appropriate pin, depending on its voltage. Mine uses 5v so I've hooked it up to the VIN pin, then the data wire to pin D5.

## Ultrasonic Distance Sensor
Last of all is the distance sensor. Give this voltage and ground, then connect the `TRIGGER` pin to D6 and `ECHO` to D7.

The wiring's complete!

# Software
Fortunately, ESPHome makes this a doddle (that's Aussie for very simple). Head over to the ESPHome tab in Home Assistant, then click the + in the bottom right-hand corner to add a new device. Give the board a name, then enter the name of your wireless network. Don't include your password yet, we'll do that later. Click `NEXT`, then select the appropriate board and click `NEXT` again. The configuration is now created.

Click `EDIT` to open the configuration. There's a couple of things we need to change here before we get going. First, on line 4, insert the name of your board. Since I'm using the NodeMCU board, I entered `nodemcuv2` (unless your NodeMCU board is very old, it's most likely a v2).

Further down, you'll see a section for wifi details. Rather than adding your password directly to this, we'll use the ESPHome `secrets` file (we'll set this up later). So, your `wifi` section should look like this:

{% highlight yml %}
wifi:
  ssid: "wifiname"
  password: !secret wifi_password
{% endhighlight %}

## Completed Configuration
With the wiring done, your `YAML` file should look something like this (except for the parts in square brackets, of course):

{% highlight yml %}
esphome:
  name: garage-door-sensor
  platform: ESP8266
  board: nodemcuv2

# Enable logging
logger:

# Enable Home Assistant API
api:

ota:
  password: "[some random string]"

wifi:
  ssid: [your wifi name]
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Garage-Door-Sensor"
    password: "[some random string]"

captive_portal:

binary_sensor:
  - platform: gpio
    name: "Garage Door Sensor"
    device_class: garage_door
    id: garage_door_sensor
    pin:
      number: D5
      inverted: True
      mode: INPUT_PULLUP

switch:
  - platform: gpio
    name: "Garage Door Opener"
    id: garage_door_switch
    pin: D1

cover:
  - platform: template
    name: "Garage Door"
    id: garage_door
    open_action:
      - switch.turn_on: garage_door_switch
      - delay: 0.5s
      - switch.turn_off: garage_door_switch
    close_action:
      - switch.turn_on: garage_door_switch
      - delay: 0.5s
      - switch.turn_off: garage_door_switch
    stop_action:
      - switch.turn_on: garage_door_switch
      - delay: 0.5s
      - switch.turn_off: garage_door_switch

sensor:
  - platform: ultrasonic
    trigger_pin: D6
    echo_pin: D7
    name: "Ultrasonic Sensor"
    update_interval: 5s
{% endhighlight %}

## Secrets File
Now, save and close your configuration. Before you install it, we need to set up the secrets file. Click the three dots in the top-right of the ESPHome dashboard, then click "Secrets Editor". Here you can add your wifi name and password, rather than inserting them into your config files directly.

The syntax is simple; it's just `variable: secret`. So, if your wifi SSID is "wireless" and your password is "esphomerules", your secrets.yaml would look like this:

{% highlight yml %}
wifi_name: wireless
wifi_password: esphomerules
{% endhighlight %}

Save that, and return to the dashboard.

# Installation
> Note: for this method, you'll need to be using a Chromium-based browser such as Chrome or Edge. It won't work on Firefox.

To install, connect your board to your computer via USB. Click `INSTALL`, then click "Plug into this computer". Select the appropriate port and wait for the installation to complete.

Now you're done!