---
layout: single
title:  "Pcapmq: Yet another Amazon Dash button hack, and many other things possible"
date:   2018-06-03 21:37:59 -0700
categories: "HomeAutomation"
header:
  overlay_image: /assets/images/laundry-dash-button_edited.jpg
  overlay_filter: 0.3
---
You just need few lines of code to hack Amazon Dash button.

# Background

I need a solution to remind me admin the medicine to my son twice a day, especially in the busy morning. So I changed my foyer's light to a Hue color, made a simple automation to change its color to yellow at 6:30 and red at 7:30. It can always remind me before we leave home. The problem is now I need a simple way to turn off the light and that automation as well after I gave him the dose. 

The medicine cabinet is on the second floor and my Echo Dot is in the kitchen, so no voice control.

The Dash button seems a perfect fit, I can put it on the cabinet, and just click it every time I finished the job. 

I know there already are tons of articles around this topic, but I don't like any of them because they are either depends on heavy scapy (such as [Dasshio](https://github.com/danimtb/dasshio)) or need another stack (such as [Dasher](https://github.com/maddox/dasher)).

So I want to go to another route, I want to have a simple program can monitor network traffic, and publish a message. I want this program is running outside of [Home Assistant](https://www.home-assistant.io), since it definitely needs root privilege, I also want to use a lightweight message, so that I can send out huge amount of messages if need (who knows how many network packets you will capture)

# A Program Publish Packet Capture Result to MQTT

I finally decided to write a python program use [pypcap](https://github.com/pynetwork/pypcap) and [hbmqtt](https://github.com/beerfactory/hbmqtt) to publish MQTT message when captured a network packet I am interest in. It has and only has two very simple features: listen the network traffic, report the result by MQTT.

Here is the repo, [https://github.com/rtfol/pcapmq](https://github.com/rtfol/pcapmq)

## Install

I published package to pypi so you can install it by `pip`, but like other solution, you have to install `libpcap` at first

```bash
sudo apt install libpython3-dev libpcap-dev
sudo pip3 install --pre pcapmq
```

## Packet Capture

`pcapmq` can be used without mqtt, it will listen the network and print the result to console. You can use following filter to capture dash button press event. Please replace `12:34:56:78:AB:CD` with your dash button's MAC address

```bash
sudo pcapmq --filter "ether src 12:34:56:78:AB:CD"  --payload-format "{} Dash button is online"
```

## Publish Message

Home Assistant has an [embedded MQTT broker](https://www.home-assistant.io/docs/mqtt/broker#embedded-broker). It make things really easy.

```bash
sudo pcapmq --filter "ether src 12:34:56:78:AB:CD"  --payload-format "ON" --broker-url mqtt://homeassistant:your_password@localhost --topic "home-assistant/dash/{1[0]:02X}.{1[1]:02X}.{1[2]:02X}.{1[3]:02X}.{1[4]:02X}.{1[5]:02X}"
```

It will connect to local mqtt broker, publish a simple ON message to the topic `home-assistant/dash/your_dash_button_mac_address`, as long as it found an Ethernet packet sent from your dash button's MAC address.

# Home Assistant Configuration

In [Home Assistant](https://www.home-assistant.io), I use a mqtt binary sensor to represent the state of dash button. Here is the `configuraiton.yml`

```yml
# MQTT
mqtt:

# Binary Sensor
binary_sensor:
  - platform: mqtt
    name: "Amazon Dash Button Sensor"
    state_topic: "home-assistant/dash/12.34.56.78.AB.CD" # use . not : to avoid potential url format issue 
    payload_on: "ON"
    payload_off: "OFF"
    device_class: moving
``` 

I have some problem to deal with json message format, so I use a simple ON message, and put the MAC address in the topic. Therefore, you can use it to listen multiple buttons

However, this sensor will never turn to OFF by itself, we have to do some automation here. In `automation.yml`

```yml
- id: '1525596293186'
  alias: Reset Dash Button
  trigger:
  - entity_id: binary_sensor.amazon_dash_button_sensor
    from: 'off'
    platform: state
    to: 'on'
  condition: []
  action:
  - delay: 0:01
  - data:
      payload: 'OFF'
      topic: home-assistant/dash/12.34.56.78.AB.CD
    service: mqtt.publish
```

Finally I use a group show my button on the home page
```yml
dash_buttons:
  name: Dash Buttons
  entities:
    - binary_sensor.amazon_dash_button_sensor
```


# Systemd

Use `systemd`, it will be an easy job to change my program into a service, here is my `/etc/systemd/system/pcapmq.service` file

```ini
[Unit]
Description=Dash button service
After=home-assistant@homeassistant.service

[Service]
Type=simple
ExecStart=/usr/local/bin/pcapmq --filter "ether src 12:34:56:78:AB:CD"  --payload-format 'ON' --broker-url mqtt://homeassistant:your_password@localhost --topic "home-assistant/dash/{1[0]:02X}.{1[1]:02X}.{1[2]:02X}.{1[3]:02X}.{1[4]:02X}.{1[5]:02X}"

[Install]
WantedBy=multi-user.target
```

Please remember to replace your button's MAC address and password, then execute following command to run the service

```bash
sudo systemctl enable pcapmq.service
sudo systemctl start pcapmq.service
```

Now, press the button!

# Alright, any question?

## Find your button's MAC
There are really million's way to find out that, for example [this one](https://github.com/danimtb/dasshio#how-to-find-the-mac-address-of-your-dash).

## Why another wheel?
I do not want to install node.js on my raspberry pi, although I use node.js for my daily job. I want to practice some python instead.

## How about duplicated message
Each dash button press will send out multiple packet, so I will send multiple message. But since I am using binary sensor, duplicate set state message is not matter at all.

## Block original function
1. You need reserved IP for dash button
2. You need set your router do block the outbound traffic from that IP

# Further Application
Next, I want to explore use `pcapmq` to do present detection.

# More Development
`pcapmq` is actually just few lines of code, I will try to do some rework next weekend to better use `asyncio` with `pypcap`

# Comments
Please leave your comments at [the home-assistant fourm](https://community.home-assistant.io/t/pcapmq-yet-another-dash-button-hack-and-many-other-things-possible/52627).
