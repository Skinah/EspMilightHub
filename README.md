# <bindingName> Binding

This is a new Openhab 2.x binding that allows a single opensource esp8266 bridge (created by Chris Mullins aka Sidoh) to automatically find and add Milight globes into OpenHab. The first question Openhab 2 users may have is “Why another binding when one already exists?”, The short answer/s to this are that the new OPENSOURCE bridge allows:

+ Almost unlimited groups so you can have individual control over an entire house of Milight globes without multiple bridges. A single bridge uses less power for one of many advantages.

+ If using the Milight remotes to control the globes, this binding will update the Openhab controls the moment a key is pressed on the remote.

+ Auto scan and adding of the globes via paper UI.

+ If you reboot Openhab2 the state of the globes will refresh and display correctly after the reboot due to the hub tracking the states and recording them in the MQTT broker.

+ Many other reasons besides just being opensource and hence can get firmware updates to support new globes and wifi KRACK patches.


## Steps to getting the hardware esp8266 Milight Hub working

In depth details on how to build and what the bridge is can be found here:
http://blog.christophermullins.com/2017/02/11/milight-wifi-gateway-emulator-on-an-esp8266

A quick overview of the steps to get the hardware going are:

+ Connect a nodemcu/esp8266 to your computer via a USB cable.
+ Download the latest BIN file from here 
<https://github.com/sidoh/esp8266_milight_hub/releases>
+ Download esp8266flasher if you are on windows 
<https://github.com/nodemcu/nodemcu-flasher>
Check the blog above on more info on how to do it from mac or linux.
+ Open the flasher tool and make sure the flash size is 4mb or whatever your esp8266 board has.
+ Flash the bin and then press the reset button on the nodemcu board when complete.
+ Connect to the wifi access point of the esp directly and setup to connect to your network. Blog has more info.
+ Login by using the IP address of the esp8266 in a web browser and the control panel will show up.
+ Connect 7 wires between the two ready made PCBs as shown in the above blog.
+ Setup a MQTT broker as this method uses the faster and lightweight MQTT protocol and not UDP.


## Steps to getting this binding running

+ Download the latest binding in a JAR format from http://www.pcmus.com/openhab/
The zip files have a date code in the format DD-MM-YYYY for when the version was built.

+ Open the zip and place the JAR file into your Openhab 'addons' folder. You do not need to install the mqtt binding as this binding is fully standalone and uses the java Paho library. You still require a MQTT broker reachable on your network, mosquitto is one example of a broker and can be installed with the Openhabian config tool, or you can use PaperUI to install the embedded MQTT broker which is found under MISC and not bindings. Do not install more than 1 broker as they can conflict.
https://www.openhab.org/addons/integrations/mqttembeddedbroker/


+ Setup the firmware of the ESP8266 using the below instructions.

+ Setup the binding using the steps covered below using either paperUI or things and items files.

## Setting up the esp8266 firmware

Enter the control panel for the ESP8266 by using any browser and entering the IP address. Follow the blog http://blog.christophermullins.com/2017/02/11/milight-wifi-gateway-emulator-on-an-esp8266  on how to setup the ESP to connect to your WIFI.

Set the following options in the firmware. Click on SETTINGS>MQTT>:

***mqtt_topic_pattern***

```
milight/commands/:device_id/:device_type/:group_id
```

***mqtt_update_topic_pattern***

leave this blank.

***mqtt_state_topic_pattern***

```
milight/states/:device_id/:device_type/:group_id
```

In the box called ***group_state_fields*** you need to untick "computed color", "brightness" and "Color", then you need to make sure the following are ticked:

+ state
+ level
+ hue
+ saturation
+ mode
+ color_temp
+ bulb_mode


Fill in the MQTT broker fields then click ***save*** down the bottom and now when you use a Milight remote control you will see MQTT topics being created that should include LEVEL and HSB. If you see brightness and not level, then go back and read the above setup steps more carefully.

You can use this linux command to watch all MQTT topics from milight:


```
mosquitto_sub -u usernamehere -P passwordhere -p 1883 -v -t 'milight/#'
```

You can also use the mosquitto_pub command to send your own commands and watch the bulbs respond. It is handy to do this if a globe I do not own does not work and you wish to request that I add a feature or fix something by giving me the mqtt message that works. Everything this binding does goes in and out via MQTT and it can be watched with the above command.


## Setting up the binding in openhab2

Just drop the JAR file into the addons folder and you should have the binding working which can now be setup with text files OR 100% with paperUI. Both methods will be covered below, but I recommend you use textual config as it is worth learning due to the speed you can make changes. Only use 1 method otherwise you will get conflict errors if trying to edit an existing thing using paperUI when the manual method has been used to define the thing.

# Key concept to understand for you to succeed

To get this working you need to know that the Milight globes are 1 way and do NOT have any kind of ID code. Only the remotes have a non editable code in them and when you LINK the globe to a remote, the globe learns this code which is referred as the remotes "Device ID". The remote has a "Group ID" of 1 to 4 if the remote supports 4 groups (some remotes support more than 4). The binding requires things to have a specific format of ID so that they can be accessed:

```
espmilighthub:type:BrokerUID:ThingUID
```

Place the Device ID and Group ID together to create the things unique ID (_ThingUID_). The manual configuration example below may make it clearer for you. The DeviceID can be in hex or decimal format but it must end with the number that is the GroupID (usually 0 for all in a group or 1 to 9 can be used). If you do not understand this key concept please post a question. 

The formula is
_ThingUID_ = DeviceID+GroupID

For example:

| Device ID | Group ID | Hex    | Decimal         |
|-----------|----------|--------|-----------------|
| 2         | 1        | 0x21   | 21              |
| 0xb4c     | 1        | 0xb4c1 | Not recommended |

If the broker thing is `espmilighthub:esp8266Bridge:1234567` and the _ThingUID_ is `0xb4c1`, the overall thing ID in OpenHAB for an `rgb_cct` bulb must be:

```
espmilighthub:rgb_cct:1234567:0xb4c1
```


## Supported Things

A bridge can have any of these things added which are the types of globes the opensource bridge supports:

+ cct
+ rgb_cct
+ rgbw
+ rgb

## Discovery

To use paperui to setup your lights, first add an EspMilightHub (aka Bridge) and then fill in the MQTT broker details in the properties of this bridge. Click on the pencil icon to reach this page where the parameters are seen. After doing this and the Bridge shows up as ONLINE you can do a scan for things and the globes should be auto found and added to your inbox if you have the auto approve and link settings ticked in paperUI’s settings.

Setting up the globes to be auto found:
For globes to be auto found you either need to use an OEM Milight remote with a linked globe, OR send a command to link the bulb first via the open source bridges control panel. Using the second method you do not need to own a physical remote and can make up any “Remote Device ID” you wish to invent. The mqtt broker saves the state when any command is sent and this allows the globe to be auto found.

To remove a globe from the saved states of your MQTT broker use this command:


```
mosquitto_pub -u username -P password -p 1883 -t 'milight/states/0x0/rgb_cct/1' -n -r
```

Replace the topic with the one you wish to remove and this will stop the globe getting autodetected by this binding.


## Binding Configuration

TODO: PR with content are welcome.

## Thing Configuration

TODO: PR with content are welcome.

## Channels

TODO: PR with content are welcome.


## Full Example

Manual configuration of the binding is my preferred method as I find it far faster to setup and also to backup. It is also handy having a list of the names and the codes that the globes are linked to.

Place the contents in a file called 'espmilighthub.things' and save it to your "things" folder.

```   
Bridge espmilighthub:esp8266Bridge:001 [ADDR="tcp://192.168.1.100:1883", MQTT_USERNAME="myusername", MQTT_PASSWORD="Suitcase123456"]
{
        Thing   rgb_cct 0xEC591 "Front Hall"    //comments are possible after double /  
        Thing   cct 0xb4c81 "Lounge Lamp 1"
        Thing   rgb_cct 0xAB13 "Linen Hall 2"       
        Thing   rgbw 20 "Bathroom Mirror All"
        Thing   rgbw 21 "Bathroom Mirror 1" //Street end
        Thing   rgbw 22 "Bathroom Mirror 2"  
        Thing   rgbw 23 "Bathroom Mirror 3"   
        Thing   rgbw 24 "Bathroom Mirror 4"          
        Thing   rgb 0xe671 "Bed2 Hall"
}
```

Additional bridge settings can be made with these:

+ FAVOURITE_WHITE
+ DELAY_BETWEEN_MQTT
+ DELAY_BETWEEN_SAME_GLOBE
+ TRIGGER_WHITE_SAT
+ TRIGGER_WHITE_HUE
+ DEFAULT_COMMAND
+ 1TRIGGERS_NIGHT_MODE
+ RGBW_WHITEMODE_SAT_THRESHOLD
+ POWERFAILS_TO_MINDIM
+ AUTOCTEMP_MAXDIMMED_TEMPERATURE


By looking in PaperUI at your bridge (click on the pencil icon) you will get descriptions on what these do and what valid ranges are. If you use manual text configuration you can not change them in paperUI otherwise you get a conflict message in paperUI. 


Example of my items file:

```   
Switch Milight_ID0xEC59_G1_State     "Light On/Off"         {channel="espmilighthub:rgb_cct:001:0xEC591:level"}
Dimmer Milight_ID0xEC59_G1_Level     "Front Hall"           {channel="espmilighthub:rgb_cct:001:0xEC591:level"}
Dimmer Milight_ID0xEC59_G1_CTemp     "White Color Temp"     {channel="espmilighthub:rgb_cct:001:0xEC591:colourtemperature"}
Color  Milight_ID0xEC59_G1_Hue    "Front Hall" ["Lighting"] {channel="espmilighthub:rgb_cct:001:0xEC591:colour"}
String Milight_ID0xEC59_G1_Cmd      "Command to Send"      {channel="espmilighthub:rgb_cct:001:0xEC591:bulbcommand"}
Switch Milight_ID0xEC59_G1_SndCmd    "Send Command"            {channel="espmilighthub:rgb_cct:001:0xEC591:sendbulbcommand"}

Switch Milight_ID2_G1_State     "Lounge Lamp 1"  ["Switchable"] {channel="espmilighthub:cct:001:0xb4c81:level"}
Dimmer Milight_ID2_G1_Level     "Brightness [%d %%]"            {channel="espmilighthub:cct:001:0xb4c81:level"}
Dimmer Milight_ID2_G1_CTemp         "White Color Temp"          {channel="espmilighthub:cct:001:0xb4c81:colourtemperature"}
```

And a sample of the sitemap contents:

```   
        Text label="EntryHallway" icon="light" 
        {

            Switch      item=Milight_ID0xEC59_G1_State
            Slider      item=Milight_ID0xEC59_G1_Level
            Slider      item=Milight_ID0xEC59_G1_CTemp
            Colorpicker item=Milight_ID0xEC59_G1_Hue
            Selection   item=Milight_ID0xEC59_G1_Cmd mappings=[next_mode='next_mode', previous_mode='previous_mode', mode_speed_up='mode_speed_up', mode_speed_down='mode_speed_down', set_white='set_white', pair='pair',unpair='unpair',level_down='level_down',level_up='level_up',temperature_down='temperature_down',temperature_up='temperature_up',night_mode='night_mode',favourite_white='favourite_white']
            Switch      item=Milight_ID0xEC59_G1_SndCmd mappings=[ON="Send"]
        
        }

            Text label="Lounge Lamp 1" icon="light" 
            {
                Switch      item=Milight_ID2_G1_State
                Slider      item=Milight_ID2_G1_Level
                Slider      item=Milight_ID2_G1_CTemp
            }
```


An example of how to group the lights so 1 control will effect any number of lights.


```
Group:Color LinenHallLights "Linen Hall" ["Lighting"]
Color LinenHall1 "First Linen Hall" (LinenHallLights) ["Lighting"] {channel="espmilighthub:rgb_cct:001:0x55442:colour"}
Color LinenHall2 "Second Linen Hall" (LinenHallLights) ["Lighting"] {channel="espmilighthub:rgb_cct:001:0x55443:colour"}

```



## Fault Finding

You can use this linux command to watch all MQTT topics from milight:


```
mosquitto_sub -u usernamehere -P passwordhere -p 1883 -v -t 'milight/#'
```


To see more detailed logs you can do this in the Openhab console:

```
log:set TRACE org.openhab.binding.espmilighthub
```

To learn how to use the Karaf console please visit this link.
https://www.openhab.org/docs/administration/console.html


change TRACE to INFO to go back to normal default output in your logs.
Whilst still in the console you can type

```
log:tail
```

And this shows the log output, CTRL + C ends it.

## How to use the light with Google Home or Alexa

The binding is setup by default for google home, so if using Alexa go into the settings of the binding for the hue and saturation triggers for white. Both need to match but any brightness value is fine. The default values 36 and 32 are for Google home, make these 0 and 100 for Alexa. Now when you ask google home or Alexa to change the light to White, it can change the globe to true white which you can select with the bindings setting called FAVOURITE_WHITE.

Example for Google Home:

```   
Bridge espmilighthub:esp8266Bridge:001 [ADDR="tcp://192.168.1.100:1883", MQTT_USERNAME="myusername", MQTT_PASSWORD="Suitcase123456", TRIGGER_WHITE_HUE=36, TRIGGER_WHITE_SAT = 32, FAVOURITE_WHITE = 200]
{
        Thing   rgb_cct 0xEC591 "Front Hall"    //comments are possible after double /  
}
```

Example for Alexa:

```   
Bridge espmilighthub:esp8266Bridge:001 [ADDR="tcp://192.168.1.100:1883", MQTT_USERNAME="myusername", MQTT_PASSWORD="Suitcase123456", TRIGGER_WHITE_HUE=0, TRIGGER_WHITE_SAT = 100, FAVOURITE_WHITE = 200]
{
        Thing   rgb_cct 0xEC591 "Front Hall"    //comments are possible after double /  
}
```

example item for both the above:

```
Color  Milight_Hue    "Front Hall" ["Lighting"] {channel="espmilighthub:rgb_cct:001:0xEC591:colour"}
```

## How to reduce the delay between globes turning on/off or other setting changes.

If you have lots of globes and openhab turns them all on, you may notice a delay that causes the globes to turn on one by one and the delay can add up when a lot of globes are installed in your house. This is caused by the time it takes to transmit the desired setting to the globe * by how many times the hub repeats transmitting the setting. Since it takes around 2.8ms for a setting to be transmitted, if the firmware is set to repeat the packets 50 times it would then take 2.8*50 = 140ms before the next globe starts to have its new state transmitted by the hub. You can reduce the packet repeats to speed up the response of this binding and the hub, but a few settings need to be considered...
 
Multiply 2.8 by your packet-repeat value (rough figure of how long it takes to transit to the globe with current firmware) and start trying the DELAY_BETWEEN_MQTT at around that figure.

Settings for the radio tab found in the esp control panel using your browser, mine are:

Packet repeats = 12 (if you only turn 1 globe on or off it uses this value)
Packet repeat throttle threshold= 200
Packet repeat throttle sensitivity = 0
Packet repeat minimum = 8 (When turning multiple globes on and off it will use this value as it throttles the repeats back to reduce latency/delay between each globe)

Only use low repeat values if your hardware is working well, I find the default value of “3” much too low for the Packet repeat minimum setting. To set this value make the two repeat settings the same so they are consistent and then see how low you can go and get reliable transmission. Packet repeats and Packet repeats minimum are key settings to play with and start off making them the same value to play with.



## How to change the lights from a rule

In an Openhab rule, you can use these commands to change the lights.

Send a desired colour in HSB format. (send to an items colour channel)

```
Milight_Hue.sendCommand("100,100,100")
```

To go back to white so long as you have not changed the bridges default settings, it is as easy as…

```
Milight_Hue.sendCommand("36,32,100")
```

Turn a globe on and then off (send to an items level channel)

```
Milight_ID0xEC59_G1_Level.sendCommand(ON)
Milight_ID0xEC59_G1_Level.sendCommand(OFF)

```

Dim a globe to 70% (send to an items level channel)


```
Milight_ID0xEC59_G1_Level.sendCommand(70)
```

## FAQ

To remove a globe from the saved states of your MQTT broker use this command:

```
 mosquitto_pub -u username -P password -p 1883 -t 'milight/states/0x0/rgb_cct/1' -n -r
```

Replace the topic with the one you wish to remove and this will stop the globe getting autodetected by this binding.
