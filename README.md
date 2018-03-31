Home Assistant configuration
============================

My configuration files and notes for [Home Assistant](https://www.home-assistant.io/) home automation platform.

My main motivation for home automation is to have some fun hacking. I'm using Hassbian running on a Raspberry Pi. I've been _trying_ to avoid using any of the Smart Hubs and stick to MQTT and WiFi devices.

Namecheap Dynamic DNS
---------------------

As I'm already using [Namechap](https://www.namecheap.com/) for a number of other projects and clients I'm using the [NamecheapDNS component](https://www.home-assistant.io/components/namecheapdns/) instead of the more commonly recommended Duck DNS.

Setting up Dynamic DNS is pretty straightforward as described in [How do I Setup a Host for Dynamic DNS](https://www.namecheap.com/support/knowledgebase/article.aspx/43/11/how-do-i-set-up-a-host-for-dynamic-dns). Before configuring Home Assistant you can easily verify that Dynamic DNS is working with a simple HTTP call.

```
https://dynamicdns.park-your-domain.com/update?host=@&domain=[YOUR_DOMAIN]]&password=[YOUR_DYNAMIC_DNS_PASSWORD]&ip=1.1.1.1
```
Once it's verified it can then be enabled in `configuration.yaml`. I _appears_ that it will update DNS at startup so I may need to setup some sort of scheduled update.

```yaml
namecheapdns:
  domain: !secret namecheap_domain
  password: !secret namecheap_dynamicdns_password
```

PositiveSSL TLS certificate
---------------------------

[Let's Encrypt](https://letsencrypt.org/) is absolutely awesome. However, I already had an unused PositiveSSL certificate in my Namecheap account. This is a simple, cheap (~$9/year) domain validated (DV) certificate.

Generate the cert with OpenSSL. The `cert.csr` file is used in the provisioning process with the CA (PositiveSSL) and the `key.pem` file is saved to the Pi to be used as the `ssl_key`.

```
openssl req -new -newkey rsa:2048 -nodes -keyout key.pem -out cert.csr
```

After validating the domain (opt of email validation or file validation, DNS is too slow) PositiveSSL sends a zip file with the certificate named `[domain].crt`. This is saved to the Pi to be used as `ssl_certificate`.

I'm storing both `key.pem` and `[domain].crt` in `/home/homeassistant/.homeassistant` as a convenience since I'm encrypting and backing up that entire folder.

The path to the key and certificate is added to the HTTP component in `configuration.yaml`.

```yaml
html:
  api_password: !secret http_password
  ssl_key: /home/homeassistant/.homeassistant/key.pem
  ssl_certificate: /home/homeassistant/.homeassistant/[domain].crt
```

Finally, the router (I'm using Google Wifi) is configured to:

1. Reserve the internal IP for the Raspberry Pi (eg. reserve 192.168.0.50 for the Pi)
2. Forward traffic on port 443 (TLS traffic) to the reserved IP (eg. 192.168.0.50) on port 8123

_Note: The Pi's firewall is setup to only allow inbound TCP traffic on port 8123_

Voila. Now Home Assistant is security externally accessible from an HTTPS URL.


Sonoff S31 Energy Monitoring Smart Plugs
----------------------------------------

This is where it gets fun! You can pickup these WiFi Smart Plug sockets from Amazon.com for about $18 each. In addition to being able to control them via WiFi (no hub needed) they also monitor energy usage.

Out of the box they have proprietary software that doesn't integrate with Home Assistant, however, you can flash the excellent open-source [Sonoff-Tasmota](https://github.com/arendst/Sonoff-Tasmota) firmware onto the S31's ESP8266 chip. This will turn them into a fully configurable smart socket that you can control via web interface, MQTT, and serial.

However, you'll need a little bit of crafty hacker skills to flash the firmware.

Do not get excited about the project that updates Sonoff software OTA (over the air). Sonoff has "fixed" the security bug that allowed that (certificate verification) and is unlikely to roll that back. To use these devices you'll need to crack the case and warm up that soldering iron.

### Preparing the Hardware

KC Budd has a great blog post on how to disassemble the Sonoff S31 and solder the 4 wires necessary to flash the ESP8266: [Sonoff S31 Disassemble and Flash Instructions](http://www.phreakmonkey.com/2018/01/sonoff-s31-disassemble-and-flash.html)

I didn't have the [SparkFun FTDI Basis Breakout - 3.3V](https://www.sparkfun.com/products/9873) so I had to wire up an FT232 I had used in another project on a breadboard per the [schematic](https://cdn.sparkfun.com/datasheets/BreakoutBoards/FTDI%20Basic-v22-3.3V.pdf).

I soldered wires coming from my breadboard directly to the VCC, RX, TX, and GND pins rather than soldering header pins onto the board. This made it quick and easy to remove the wires after the flash was complete.

![Soldering wires to Sonoff S31 to flash firmware](img/sonoff-s31-firmware-solder-wires.png?raw=true)

### Preparing the Firmware

After [installing PlatformIO for Atom](https://platformio.org/get-started/ide?install=atom), clone the [Sonoff-Tasmota](https://github.com/arendst/Sonoff-Tasmota) repository and open that folder in Atom.

Two files need to be edited before building and uploading the firmware to the S31.

In the project root is a file named `platformio.ini`. Simply un-comment default environment. `sonoff` is the EN environment but there are other languages available.

![PlatformIO specify environment](img/platformio-sonoff-environment.png?raw=true)

The `sonoff/user_config.h` header file defines some _initial_ configuration for the firmware. The firmware does allow much of the configuration to be changed via the web interface or MQTT once it's been flashed so the most important settings are the WIFI and MQTT settings.

![PlatformIO user configuration](img/platformio-sonoff-user-config-h.png?raw=true)

* **PROJECT** - This ends up being the device name as well as a unique identifier in the MQTT topic. I'm using a description of what will be plugged into the S31 like "Coffee_Maker" or "Washing_Machine".
* **STA_SSID1** - The SSID of the WiFi network.
* **STA_PASS1** - The password to connect to the WiFi network.
* **MQTT_HOST** - The host or IP of the MQTT broker. In my case it is the Raspberry Pi running Mosquitto.
* **MQTT_PORT** - The MQTT broker port.
* **MQTT_USER** - The MQTT username. Don't allow an unauthenticated MQTT broker.
* **MQTT_PASS** - The MQTT password. Don't allow an unauthenticated MQTT broker.

*Pro Tip: If you're not using DHCP on your network you'll need to configure the WIFI_XXX settings as well.*

*Pro Tip: If you change this configuration after flashing the device you'll need to update the CFG_HOLDER value for it to take effect the next time you flash.*

Select the `PlatformIO => Build` menu item to build the firmware.

Success looks something like:

```
Linking .pioenvs/sonoff/firmware.elf
Calculating size .pioenvs/sonoff/firmware.elf
Building .pioenvs/sonoff/firmware.bin
text       data     bss     dec     hex filename

493803     9268   41528  544599   84f57 .pioenvs/sonoff/firmware.elf
========= [SUCCESS] Took 7.68 seconds =========
```

*Pro Tip: if the build window disappears too quickly after completion press F8 to toggle it back into view.*


### Flashing the Firmware

Flashing the firmware of the S31 consists of putting the ESP8266 into "program mode" and then uploading the firmware over a serial connection via the FT232 chip. The ESP8266 chip will be powered by the 3.3V out from the FT232 and the FT232 will be powered by the 5V out from the USB port of the computer.

**DO NOT PLUG THE S31 INTO THE AC OUTLET DURING THIS PROCESS**

In order to put the ESP8266 chip into program mode, the button on the S31 needs to be held down while being plugged into the USB port. The button can be held down for a second or two after the USB cable is plugged in for good measure.

Once in program mode, the firmware can be uploaded by selecting the `PlatformIO => Upload` menu item.

Success looks something like this:

```
Building .pioenvs/sonoff/firmware.bin
Configuring upload protocol...
Looking for upload port...
Auto-detected: /dev/ttyUSB0
Uploading .pioenvs/sonoff/firmware.bin
Uploading 507216 bytes from .pioenvs/sonoff/firmware.bin to flash at 0x00000000
........................................................ [ 16% ]
........................................................ [ 32% ]
........................................................ [ 48% ]
........................................................ [ 64% ]
........................................................ [ 80% ]
........................................................ [ 96% ]
.........                                               [ 100% ]
================= [SUCCESS] Took 65.72 seconds =================
```

Troubleshooting can be tricky if something goes wrong. Here are things to triple check:

* Repeat the process of putting the device into program mode and flashing 2-3 times before assuming something is wired incorrectly.
* Make sure the S31 PCB is getting 3.3V and not 5V on the VCC pin.
* Make sure the connections between the FT232 and the S31 PCB are RX => TX and TX => RX.
* Make sure PlatformIO is auto-detecting the right USB port (notice the "Auto-detected" line in the above success output).
* Try swapping the RX and TX wires just to be sure if it keeps failing.
* Make sure the button on the S31 PCB was pressed _before_ plutting in the USB cable.
* Put LEDs with 1k pull-up resistors on the LED pins of the FT232 to confirm data transfer.
* Re-check all the connections.

Immediately after flashing the Sonoff S31 it will be configurable via MQTT, web interface, and serial. Select the `PlatformIO => Serial Monitor` menu item to confirm the device is able to create the webserver and start publishing on MQTT.

```
--- Miniterm on /dev/ttyUSB0  115200,8,N,1 ---
--- Quit: Ctrl+C | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H ---
␀
00:00:07 WIF: Connected
00:00:07 HTP: Web server active on Entertainment_Center_Socket-2088 with IP address 192.168.86.103
00:00:07 MQT: Attempting connection...
00:00:08 MQT: Connected
00:00:08 MQT: tele/Entertainment_Center_Socket/LWT = Online
00:00:08 MQT: cmnd/Entertainment_Center_Socket/POWER =
00:00:08 MQT: tele/Entertainment_Center_Socket/INFO1 = {"Module":"Sonoff Basic","Version":"5.12.0i","FallbackTopic":"DVES_7D2828","GroupTopic":"sonoffs"}
00:00:08 MQT: tele/Entertainment_Center_Socket/INFO2 = {"WebServerMode":"Admin","Hostname":"Media_Center-2088","IPAddress":"192.168.86.103"}
00:00:08 MQT: tele/Entertainment_Center_Socket/INFO3 = {"RestartReason":"Power on"}
00:00:09 MQT: stat/Entertainment_Center_Socket/RESULT = {"POWER":"OFF"}
00:00:09 MQT: stat/Entertainment_Center_Socket/POWER = OFF
```

The device is ready to be configured.

### Configuring the Sonoff S31 Firmware

The Sonoff S31 will initially be configured with the *Sonoff Basic* module. In order to use the energy monitoring sensors it will need to be configured with the *Sonoff S31* module.

Navigate a web browser to the IP address listed in the serial output (`PlatformIO => Serial Monitor`) line:

```
00:00:07 HTP: Web server active on Media_Center-2088 with IP address 192.168.86.103
```

This will bring up the web interface.

![Sonoff Basic Web Interface](img/sonoff-s31-web-configuration.png?raw=true)

Notice the "Sonoff Basic Module" title. Click the `Configuration` button and then `Configure Module` button. Select the module type `41 Sonoff S31` from the drop-down menu and click `Save`.

Navigate back to the main menu and you'll now see the Sonoff S31 energy values.

![Sonoff S31 Module Web Interface](img/sonoff-s31-module-configured.png?raw=true)

Click the `Console` button. This view shows the MTQQ messages being published by the device and also allows you to enter [Sonoff-Tasmota Firmware Commands](https://github.com/arendst/Sonoff-Tasmota/wiki/Commands#sonoff-pow-sonoff-s31-and-pzem004t) that can return information or change configuration values.

Take note of this, as these MQTT topics are what are used in the Home Assistant configuration later.

![Sonoff S31 Module Web Interface](img/sonoff-s31-configure-console.png?raw=true)

There are two more settings that are important for integrating with Home Assistant. Type `PowerRetain 1` into the command input box and press `Enter`. Then type `SensorRetain 1` into the command input box and press `Enter`.

These _retain_ settings tell the MQTT broker that when a new client connects to the topic it should send get the last value so it can initialize it's state. What this means is that when Home Assistant connects to the MQTT topic for the Sonoff S31 it will get the power state (on/off) and the sensor states (volgate, current, power, etc.). This becomes important for interacting with the Sonoff S31 in the Home Assistant GUI.


### Configuring S31 Home Assistant Switches

The Sonoff S31 has a button to turn the power to the plugged in appliances on and off. This can be controlled and monitored as a switch in Home Assistant.

![Sonoff S31 Switches in Home Assistant](img/sonoff-s31-home-assistant-switches.png?raw=true)

`configuration.yaml`

```yaml
switch:
  - platform: mqtt
    name: Entertainment Center
    command_topic: "cmnd/Entertainment_Center_Socket/POWER"
    state_topic: "stat/Entertainment_Center_Socket/POWER"
    availability_topic: "tele/Entertainment_Center_Socket/LWT"
    qos: 1
    payload_on: "ON"
    payload_off: "OFF"
    payload_available: "Online"
    payload_not_available: "Offline"
    retain: true
```

The `cmnd/<PROJECT_NAME>/POWER` topic is how Home Assistant will send a command to the S31 to turn it on or off.

The `stat/<PROJECT_NAME>/POWER` topic notifies Home Assistant what the current state of the S31 is (ON or OFF).

The `stat/<PROJECT_NAME>/LWT` topic notifies Home Assistant if the device is offline (LWT = Last Will and Testament). This tells Home Assistant to render a "Offline" or "Unavailable" state in the GUI rather than a toggle switch that can be interacted with. Unplugging the device from the wall is one scenario which will result in this state.

The `payload_XXX` keys tell Home Assistant how to send/read the values in the MQTT topics to/from S31.


### Configuring S31 Home Assistant Sensors

The Sonoff S31 includes a number of energy monitoring sensors which can be integrated into Home Assistant.

The sensor data is found on the `tele/<PROJECT_NAME>/SENSOR` as a JSON object containing power, power factor, voltage, current, and energy consumption for the day, the previous day, and in total.

The JSON payload in the MQTT message is shown below:

```JSON
{
  "Time":"2018-03-31T01:55:55",
  "ENERGY": {
    "Total":0.349,
    "Yesterday":0.145,
    "Today":0.204,
    "Period":9,
    "Power":105,
    "Factor":0.92,
    "Voltage":122,
    "Current":0.927
  }
}
```

The JSON payload can be parsed into individual sensors in the Home Assistant `configuration.yaml`.

```yaml
sensor:
  - platform: mqtt
    name: Entertainment Center Voltage
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Voltage }}"
    unit_of_measurement: "V"
  - platform: mqtt
    name: Entertainment Center Current
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Current }}"
    unit_of_measurement: "A"
  - platform: mqtt
    name: Entertainment Center Power
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Power }}"
    unit_of_measurement: "W"
  - platform: mqtt
    name: Entertainment Center Energy Today
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Today }}"
    unit_of_measurement: "kWh"
  - platform: mqtt
    name: Entertainment Center Energy Yesterday
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Yesterday }}"
    unit_of_measurement: "kWh"
  - platform: mqtt
    name: Entertainment Center Energy Total
    state_topic: "tele/Entertainment_Center_Socket/SENSOR"
    value_template: "{{ value_json['ENERGY'].Total }}"
    unit_of_measurement: "kWh"
```

![Sonoff S31 Sensor Badges in Home Assistant](img/sonoff-S31-home-assistant-sensor-badges.png?raw=true)

![Sonoff S31 Sensors in Home Assistant](img/sonoff-s31-home-assistant-sensors.png?raw=true)
