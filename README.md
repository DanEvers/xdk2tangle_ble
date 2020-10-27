# XDK2MAM Bluetooth Low Energy (BLE) WITH SD CARD
The XDK provides several BLE APIs to manage the BLE functionality on the device. The interfaces can be used by applications in order to communicate via the ALPWISE BLE stack with surrounding BLE devices.
A wide range of BLE functionalities can be achieved using the XDK, from configuration of the BLE controller according to the requirements of the designed XDK application, up to active BLE connections, including data exchange in both ways between the XDK and other BLE devices. 

The following repository has either files for the Bosch XDK 110 and for the data receiver in Rust where the attach to Tangle via Streams happens. 

**This package uses a config file on a micro sd card, which makes possible to alter some values as the interval or the sensors used without need to recompile (you just change values in the config file and you are ready to go)**

# Instructions for the XDK110

## Requirements
In order to be able to run the code on this repo you will to [download XDK Workbench](https://developer.bosch.com/web/xdk/downloads), have a XDK 110 and install Node on the computer you are going to use as listener server.

## Flashing your XDK: wifi and sensors configuration
Open XDK Workbench and go to File -> Import. Choose General > Projects from Folder or Archive and select the folder ***iot2tangle-c***. Accept to import project. 

### Clear, Build and Flash
Open XDK Workbench and go to File -> Import. Choose General > Projects from Folder or Archive and select the folder **XDK110-Bosch/http-sdcard**. Accept to import project. Once project is imported, right click on **iot2tangle** folder in your Workbench Project Explorer and select **Clean project**. When the clean is done, repeat and select **Build Project**. This process can take some minutes depending on your hardware and you should see any problems at the Workbench Console.

Finally, once the project has been built, connect your XDK 110 via USB and click the ***Flash*** button to install the software on the board. If everything went fine, you should be able to see the sensor data on your console.

### Editing config data

Open the **config.cfg** file on your computer and change the values to match your WLAN data, host, port and the sensors you want to use.

```
DEVICE_NAME=enter-your-device-id
INTER_REQUEST_INTERVAL=30000
INTERVAL_STREAM_DIVIDER_BLE=250
ENVIROMENTAL=YES
ACCELEROMETER=YES
GYROSCOPE=YES
INERTIAL=YES
LIGHT=YES
MAGNETOMETER=YES
ACOUSTIC=YES
```

Save the values, extract the micro SD card and carefully insert it into the XDK SD slot (contacts up). 
Turn on the XDK and you are good to go! 
If everything went fine the XDK110 should now be sending its sensors data to the given destination server. 

### Dealing with the Invalid application error
Some XDK110 using the 1.1.0 bootloader version produce an invalid application output when the flash process finishes. If you get this error, try clicking on the **Boot** button (it should reboot and give the error again) and then click again on **Flash**. If you get the error again, repeat the process. We don't know why this error is produced and have already informed the XDK110 team at Bosch about it.


# Setting up the Streams Gateway

**Note:** you can run the Gateway on a Raspberry Pi, a local Node in your Network or a VPS


## Preparation

Install Rust if you don't have it already, find the instructions here https://www.rust-lang.org/tools/install

`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

Make sure you also have the build dependencies installed, if not run:  

`sudo apt install build-essential`  
`sudo apt install pkg-config`  
`sudo apt install libssl-dev`  
`sudo apt update`  

## Installing the IOTA Stream Gateway

Get the IOTA Streams WiFi Gateway. 

`git clone https://github.com/iot2tangle/Streams-http-gateway`

Navigate to the **Streams-wifi-gateway** directory and edit the **config.json** file to define your device name (it must match what you set on the Sense Hat config).
There you can also change ports and the IOTA Full Node used.  
  
```
{
    "device_name": "XDK-BLE", 
    "port": 8080, 
    "node": "https://nodes.iota.cafe:443", 
    "mwm": 14,    
    "local_pow": false     
}
```

**Set the *device_name* to the value specified in the XDK110 configuration file as DEVICE_NAME**  
Change *port, node, mwm, local_pow* if needed. 

## Start the Streams Server

### Sending messages to the Tangle

Run the Streams Gateway:  

`cargo run --release`  

This will compile and start the Streams Gateway. Note that the compilation process may take from 3 to 30 minutes (Pi3 took us around 30 mins, Pi4 8 mins and VPS or desktop machines will generally compile under the 5 mins) depending on the device you are using as host.

You will only go through the compilation once and any restart done later will take a few seconds to have the Gateway working.

![Streams Gateway receiving SenseHat data](https://iot2tangle.io/assets/screenshots/PiSenseHatSend.png)
*The Gateway starts by giving us the channel id that will allow subscribers to access the channel data.*

### Reading messages from the Tangle

In a separate console start a subscriber using the Channel Id printed by the Gateway (see example above):  

`cargo run --release --example subscriber <your_channel_root> `  

![Streams Gateway receiving SenseHat data](https://iot2tangle.io/assets/screenshots/PiSenseHatGet.png)


### Testing the gateway without sensors

To send data to the server you can use Postman, or like in this case cURL, make sure the port is the same as in the config.json file:  
`  
curl --location --request POST '127.0.0.1:8080/sensor_data'   
--header 'Content-Type: application/json'   
--data-raw '{
    "iot2tangle": [
        {
            "sensor": "Gyroscope",
            "data": [
                {
                    "x": "4514"
                },
                {
                    "y": "244"
                },
                {
                    "z": "-1830"
                }
            ]
        },
        {
            "sensor": "Acoustic",
            "data": [
                {
                    "mp": "1"
                }
            ]
        }
    ],  
    "device": "XDK_BLE",  
    "timestamp": "1558511111"  
}'  
`   
IMPORTANT: The device will be authenticated through the "device" field in the request (in this case XDK-BLE), this has to match what was set as device_name in the config.json on the Gateway (see Configuration section above)!  
  
After a few seconds you should now see the data beeing recieved by the Subscriber!
