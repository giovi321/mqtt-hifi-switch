# MQTT HiFi Switch
A simple script in bash that turns on a hifi system when music is being played on a raspberry through airplay or bluetooth using Home Assistant.

## Context
* I use an amplifier in my livingroom to play music from my smartphone;
* As "connection" I use a raspberry running one of the many solutions available to have an Apple AirPlay server equivalent (see [this](https://www.google.com/search?q=raspberry+airplay) for the software and [this](https://www.google.com/search?q=raspberry+toslink+audio) for the hardware);
* My stereo is connected to a wifi smart-plug that is controlled from my ha server, so that i can turn on and off the stereo from ha.

## Problem
* I always forget to turn off the stereo ending up wasting a lot of energy;
* I'm lazy and I don't want to turn on the stereo through home-assistant every time I want to listen to music.

## Solution
* I set-up a ~5 lines script on the raspberry running the AirPlay server - it published on a MQTT topic a message stating whether the raspberry is playing any music or not;
* On ha I set up a simple automation to:
    * turn on the stereo if it is off and there is music playing;
    * turn off the stereo if no music is playing for >10 minutes.

## How-to
Begin the set-up on the raspberry running the AirPlay server:
```
apt install mosquitto-clients
nano /root/checkaudio.sh
```
Content of checkaudio.sh:
```
#!/bin/bash
if [[ $(cat /proc/asound/card0/pcm0p/sub0/status | grep state) = "state: RUNNING" ]]
then
    mosquitto_pub -h your_mqtt_broker_ip --topic "/home-assistant/stereoplaying" -m "1" --username your_mqtt_broker_username --pw your_mqtt_broker_password
else
    mosquitto_pub -h your_mqtt_broker_ip --topic "/home-assistant/stereoplaying" -m "0" --username your_mqtt_broker_username --pw your_mqtt_broker_password
fi
```
Continue with the set-up:
```
chmod 655 checkaudio.sh
nano /etc/systemd/system/checkaudio@root.service 
```
Content of checkadio@root.service:
```
[Unit]
Description=Check audio

[Service]
Type=oneshot
ExecStart=/root/checkaudio.sh
```
Continue with set-up:
```
nano /etc/systemd/system/checkaudio@root.timer
```
Content of checkaudio@root.timer:
```
[Timer]
OnUnitActiveSec=1s
AccuracySec=1ms
Unit=checkaudio@root.service
OnBootSec:1
```
Continue with set-up:
```
systemctl enable checkaudio@root.timer
```
Now reboot your system:
```
shutdown -r now
```
You chan check whether the mqtt message is being publihed on the topic specified (i.e., whether the script is really telling to home-assistant "hey I'm playing something" or not) issuing this command on the machine where the sound is playing:
```
mosquitto_sub -h your_mqtt_broker_ip --topic "/home-assistant/stereoplaying" --username your_mqtt_broker_username --pw your_mqtt_broker_password
```
If there is sound being played, you should continuously see "1" appear on the output, otherwise if no sound is playing you should see "0".

Now it's time to add the component to home-assistant configuration. Contents to add to configuration.yaml:
```
binary_sensor:
   platform: mqtt
   name: "Stereo playing"
   state_topic: "/home-assistant/stereoplaying"
   payload_on: "1"
   payload_off: "0"
```

Set-up the automations you prefer. I have two automations: i) turn off the stereo if nothing is playing for 10 minutes, ii) turn on the stereo as soon as something is playing. I created the two automations through the new automations interface:

```
- id: 'xxxxxxx'
  alias: Turn off stereo if music not playing
  trigger:
  - entity_id: binary_sensor.stereo_playing
    for:
      minutes: 10
    from: 'on'
    platform: state
    to: 'off'
  action:
  - data:
      entity_id: group.stereo
    service: homeassistant.turn_off
- id: 'xx'
  alias: Turn on stereo if music playing
  trigger:
  - entity_id: binary_sensor.stereo_salotto_playing
    from: 'off'
    platform: state
    to: 'on'
  action:
  - alias: ''
    data:
      entity_id: group.stereo
    service: homeassistant.turn_on
```
Now you just need to restart your home-assistant instance.

![5efa446e3cef3433e11957c2afafa854c8e27e0e](https://github.com/giovi321/mqtt-hifi-switch/assets/6443515/c3d9c0e5-620f-4fd3-889c-7ff11a3cc979)

