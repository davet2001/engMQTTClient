# engMQTTClient
MQTT client for Energenie [ener314-rt board] (https://energenie4u.co.uk/catalogue/product/ENER314-RT) running on a raspberry Pi

Please note that this is for the 'two way' ener314-rt board, which is different from 'one way' ener314 board, and does not work for the 'one way' board.

![ener314-rt board] (https://energenie4u.co.uk/res/images/products/medium/RXTX Board Front L.jpg)

engMQTTClient uses the mosquitto MQTT client library to do the MQTT handling.  Communication with the Energenie products is developed from the code in the energenie example directory eTRV/eTRV\_TX/HopeRF-TX-RX

It also uses the [log4c] (http://log4c.sourceforge.net/) logging library and the [cJSON] (https://sourceforge.net/projects/cjson/) parser.

## Status
Alpha.  It works for me. Your experience may vary.

Working : 
* sending ON/OFF commands to ENER002 remote controlled sockets.
* MIH0013 (eTRV) commands 
Commands are sent using topics /energenie/eTRV/Command/_Command_/sensorId
where _Command_ can be

| Command | Payload | Comment |
|--------|---------|---------|
| Temperature | 4-30 (Ascii) | Set Target Temperature
| Identify | None     | Makes the MIH0013 flash |
| Exercise | None     | Causes Valve to go up and down.  Creates Diagnostic when finished.
| Voltage  | None     | MIH0013 reports battery Voltage
| Diagnostics | None   | Causes MIH0013 to report diagnostics
| ValveState | "0", "1" or "2" (ascii) | 0=Set Valve Open 1=Closed 2=Normal Operation
| PowerMode  | "0", "1" | 0=Low Power Mode off 1=On
| ReportingInterval | 300-3600 (ascii) | Set the Reporting Interval (not tested)

* MIH0013 (eTRV) reports
Reports are received on Topic /energenie/eTRV/Report/_Report_/sensorId
where _Report_ can be

| Report | Payload | Comment |
|--------|---------|---------|
| Temperature | Ascii string | Measured Temperature in degrees Centigrade
| TargetTemperature | Ascii String | Target Temperature set
| Diagnostics | 2 bytes | byte 0 = low byte, 1 = high byte
| Voltage | Ascii String | Reported Battery Voltage

## Building

### Log4c
The version from the raspian respository is too old to work, so you need to get the tarball and install that.  See the instructions at [http://log4c.sourceforge.net/#installation]

### Mosquitto
Install the following packages

	pi@raspberrypi ~/energenie/engMQTTClient $ dpkg --get-selections | grep mosquitto
	libmosquitto-dev:armhf                          install
	libmosquitto1:armhf                             install
	mosquitto                                       install
	mosquitto-clients                               install

### engMQTTClient

Clone this repository and compile engMQTTClient using 'make'.

Run the program using

        sudo LD_LIBRARY_PATH=/usr/local/lib ./engMQTTClient

assuming log4c has been placed in /usr/local/lib as per default.

### MQTT Topic structure

For ENER002 sockets, using its own protocol the structure is
        /energenie/ENER002/_address_/_socketnum_

        _address_ is 2, 8 or 16 which is used to generate the OOK address.  The algorithm to produce this is specified in the sample code.  This changes the first digit to another that I found works (the original value in the sample code is 8).  This will enable use for up to 12 sockets, which is enough for now.
        _socketnum_ is 0-4.  0 addresses all the sockets accessed by address, 1-4 access individual sockets at that address.

For eTRV (and hopefully other OpenThings protocol devices) the structure to send command to device is
        /energenie/eTRV/Command/_commandid_/_deviceid_

To receive commands the structure is
        /energenie/eTRV/Report/_commandid_/_deviceid_

        _commandid_ is the command sent or received (so far "Identity" or "Temperature")
        _deviceid_ is the openThing id number for the device in decimal
        
## Usage

### Command Line Example

Listen for Incoming eTRV reports
```sh
mosquitto_sub -v -h your_mosquitto_broker -t /energenie/#
```

Turn on socket
```sh
mosquitto_pub -h your_mosquitto_broker -t /energenie/ENER002/8/4 -m On
```

Identify eTRV (flashes led for 60s)
```sh
mosquitto_pub -h your_mosquitto_broker -t /energenie/eTRV/Command/Identify/329 -n
```

### Python Example

Install paho library
```
sudo pip install paho-mqtt
```

Turn Socket On
```Python
        import paho.mqtt.publish as publish

        # Switch On
        publish.single("/energenie/ENER002/8/4","On", hostname="192.168.0.3")
```

Listen for eTRV temperature reports (can also be used to find out the address of your eTRVs)
```Python
	import paho.mqtt.client as mqtt
	import subprocess
	import time

	broker_address = "192.168.0.3"
	broker_port = 1883

	def on_connect(client, userdata, flags, rc):
    		client.subscribe("/energenie/eTRV/Report/Temperature/#")

	def on_message(client, userdata, msg):
    		print (msg.topic+" "+str(msg.payload)+ "C")

	client = mqtt.Client()
	client.on_connect = on_connect
	client.on_message = on_message
	client.connect(broker_address, broker_port, 60)
	client.loop_forever()

```
	
### OpenHab example

#### Controlling switches

Item file

        Switch Light_FF_Bed_Table 	"Bedside Lamp" 	(FF_Bed, Lights)
         {mqtt=">[raspberryPI:/energenie/ENER002/8/1:command:ON:On],>[raspberryPI:/energenie/ENER002/8/1:command:OFF:Off]"}

Sitemap file
        
        Switch item=Light_FF_Bed_Table label="Bedroom Light" icon="switch"

#### Controlling eTRV

Item file

        Group Temperature_Chart 
        Number Temperature_329_set    "Lounge Temperature Target [%.1f °C]" <temperature> (Temperature_Chart, FF_Bed)   
                                      {mqtt=">[raspberryPi:/energenie/eTRV/Command/Temperature/329:command:*:${command}]"}
        Number Temperature_329        "Lounge Temperature [%.1f °C]"   <temperature> (Temperature, Temperature_Chart, FF_Bed)
                                      {mqtt="<[raspberryPi:/energenie/eTRV/Report/Temperature/329:state:default"}

        Number Temperature_Chart_Period		"Chart Period"

        Number eTRV_329_received_target   "Lounge Received Target" <temperature> (Trv, FF_Bed, Temperature_Chart)
                                          {mqtt="<[raspberryPi:/energenie/eTRV/Report/TargetTemperature/329:state:default"}
        Number eTRV_329               "Lounge TRV Control"    (Trv, FF_Bed)
                                          {mqtt=">[raspberryPi:/energenie/eTRV/Command/Identify/329:command:1:0]"}
                                          
Sitemap file

        Setpoint item=Temperature_329_set icon="temperature" minValue=4 maxValue=30 step=1
	Text item=Temperature_329
	Switch item=eTRV_329 mappings=[1="Identify"]
		
	Switch item=Temperature_Chart_Period label="Chart Period" mappings=[0="Hour", 1="Day", 2="Week"]
	Chart item=Temperature_Chart period=h refresh=6000 visibility=[Temperature_Chart_Period==0, Temperature_Chart_Period=="Uninitialized"]
	Chart item=Temperature_Chart period=D refresh=30000 visibility=[Temperature_Chart_Period==1]
	Chart item=Temperature_Chart period=W refresh=30000 visibility=[Temperature_Chart_Period==2]
	
### Home Assistant example

In configuration.yaml

```
light:
  - name: "Bedside Lamp"
    platform: mqtt
    command_topic: "/energenie/ENER002/8/1"
    payload_on: "On"
    payload_off: "Off"

sensor lounge:
  - platform: mqtt
    state_topic: "/energenie/eTRV/Report/Temperature/329"
    name: "Lounge Temperature"
    unit_of_measurement: "°C"

```

## License
This code is published under the MIT License.  The last paragraph is important.  If running this code causes your device to fail, I'm not responsible.

The mosquitto MQTT library is released under the Eclipse Public License.

The log4c logging library is released under the LGPL license.

The cJSON JSON parser is released under the MIT license.
