# Using Node-RED with the Sensor Hub

* [Installation](#Installation)
* [Creating your first flow](#Creatingyourfirstflow)
	* [1. Add an Inject node](#AddanInjectnode)
	* [2. Add a Debug node](#AddaDebugnode)
	* [3. Wire the two together](#Wirethetwotogether)
	* [4. Deploy](#Deploy)
	* [5. Add a Function node](#AddaFunctionnode)
* [Using with MQTT on the Hub](#UsingwithMQTTontheHub)
	* [Configuring the internal broker](#Configuringtheinternalbroker)
	* [Subscribing to topics](#Subscribingtotopics)
	* [Publishing to topics](#Publishingtotopics)
* [Example: Smart Home Integration using MQTT](#SmartHomeIntegrationusingMQTT)
	* [Installing 3rd-party nodes](#Installing3rd-partynodes)
	* [Google Home (NORA)](#GoogleHomeNORA)
	* [Amazon Alexa](#AmazonAlexa)
* [End](#End)
	* [Resources](#Resources)


## <a name='Installation'></a>Installation

Run the following command to install:

```linux
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```

After installation, to run as a service:

```linux
node-red-start
```

To run on startup:

```linux
sudo systemctl enable nodered.service
```

Once Node-RED is running, you can access the editor from another machine with the IP address of the hub: `http://<ip-address>:1880`

More information can be found here: https://nodered.org/docs/getting-started/raspberrypi

## <a name='Creatingyourfirstflow'></a>Creating your first flow

### <a name='AddanInjectnode'></a>1. Add an Inject node

The `inject` node allows you to inject preset messages into a flow, either by clicking the button, or setting a time interval between injects.

Drag one onto the workspace (main grid) from the palette (left sidebar).

![inject](https://i.imgur.com/AVjgXjp.png)

### <a name='AddaDebugnode'></a>2. Add a Debug node

The `debug` node allows any message to be displayed in the Debug sidebar (right) which can be accessed from here:

![debug](https://i.imgur.com/hZ65wwI.png)

### <a name='Wirethetwotogether'></a>3. Wire the two together

Connect the `inject` and `debug` nodes by dragging from the output port of one to the input port of the other.

![wire](https://i.imgur.com/tgrrgXS.png)

### <a name='Deploy'></a>4. Deploy

Click the Deploy button (top-right) to deploy the flow to the server. For each press of the inject button on the `inject` node, in the Debug sidebar you should see a number pop up (timestamp in milliseconds).

### <a name='AddaFunctionnode'></a>5. Add a Function node

The `function` node allows you to pass messages through a JavaScript function.

Delete the existing wire (select and press Delete on keyboard).

Wire a `function` node between the `inject` and `debug` nodes. Double-click on the `function` node to bring up the edit dialog. Enter the following code into the function field:

```javascript
// Create a Date object from the payload
var date = new Date(msg.payload);
// Change the payload to be a formatted Date string
msg.payload = date.toString();
// Return the message so it can be sent on
return msg;
```

Click Done to close the edit dialog and deploy the flow.

Now when you click the inject button, the function will take in the raw timestamp from the `inject` node, format it as a date, and send it to the `debug` node to be logged in the Debug sidebar.







## <a name='UsingwithMQTTontheHub'></a>Using with MQTT on the Hub

### <a name='Configuringtheinternalbroker'></a>Configuring the internal broker

Add an `mqtt in` node and double-click it to edit the properties. Add a new broker with the following settings:

![broker](https://i.imgur.com/Ev8Sdng.png)

Then, go to the Security tab and enter the username and password for the internal broker.

### <a name='Subscribingtotopics'></a>Subscribing to topics

Configure an `mqtt in` node to subscribe to the `events/object/rawTemperature` topic:

![subscribe](https://i.imgur.com/YK5xrVO.png)

Connect this to a `debug` node and you should see the temperature being logged when you deploy the flow. Any time a change in temperature is observed, it should be logged as well.

![subscribe2](https://i.imgur.com/UGLVxNe.png)

### <a name='Publishingtotopics'></a>Publishing to topics

Configure an `mqtt out` node to publish to the `commands/object/lightringPattern` topic:

![publish](https://i.imgur.com/iYu78du.png)

Connect this to an `inject` node with `msg.payload` set to a JSON value. Enter the following as the value:

```json
{
    "data": "2"
}
```

Now deploy the flow. When you press the inject button you should see the 2nd light ring pattern (blue swirl) light up on the sensor hub. If nothing is showing up, ensure that your login for the internal broker was entered correctly.

![publish2](https://i.imgur.com/2CEw4PH.png)

## <a name='SmartHomeIntegrationusingMQTT'></a>Example: Smart Home Integration using MQTT

### <a name='Installing3rd-partynodes'></a>Installing 3rd-party nodes

For this example, we will be using 3rd-party nodes to handle the integration with Google Home and Amazon Alexa.

Select the `Manage Palette` option from the main menu (top-right) to open the Palette Manager. Go to the Install tab and install the following packages:

- node-red-contrib-nora
- node-red-contrib-alexa-smart-home

![install](https://i.imgur.com/vrJvBf3.png "Ex: Installing node-red-contrib-nora")

This can take a few minutes. Click `View log` to check the status.

### <a name='GoogleHomeNORA'></a>Google Home (NORA)

We will be adding the Sensor Hub as a thermostat to our Google Home using the NORA nodes that we added earlier.

Add a nora `thermostat` node to your flow, go into the node's properties, and edit the Config (nora config). Sign in [here](https://node-red-google-home.herokuapp.com/) to get a token and copy it into your nora config.

With the node selected, you can view the acceptable inputs in the Help sidebar:

![nora1](https://i.imgur.com/buNSw13.png)

Create an `inject` node for the thermostat's mode. Connect it to the `thermostat` input. Be sure "Inject once after 0.1 seconds" is checked so that it is set on startup. Set `msg.payload` to a JSON value. Enter the following as the value:

```json
{
    "mode": "cool"
}
```

Create `mqtt in` nodes for setpoint (topic: `events/object/tempSetPoint`) and temperature (topic: `events/object/rawTemperature`).

At this point, your flow should look like this:

![nora2](https://i.imgur.com/tRWuhWB.png)

As you saw previously, the `mqtt in` node outputs something like:

```json
{
  "Present_Value": 25.32,
  "Units": "°C",
  "updated": "31-07-2020 02:18:43.0",
  "status": 0
}
```

But the `thermostat` would expect this as its input (for temperature):

```json
{
    "temperature": 25.32
}
```

So we can add a `function` node between the `mqtt in` and `thermostat` nodes to handle this. The code is as follows for the temperature:

```javascript
var jsonPayload = JSON.parse(msg.payload);
var temperature = jsonPayload.Present_Value;

return {"payload": {"temperature": temperature}};
```

And for the setpoint:

```javascript
var jsonPayload = JSON.parse(msg.payload);
var setpoint = jsonPayload.Present_Value;

return {"payload": {"setpoint": setpoint}};
```

You can also add an `mqtt out` node for the setpoint (topic: `events/object/tempSetPoint`). Add a `function` node to convert the `thermostat` output to what the MQTT topic expects. The code is as follows:

```javascript
var jsonPayload = JSON.parse(msg.payload.setpoint);

return {"payload": {"Present_Value": jsonPayload, "Units": "", "updated":  "31-07-2020 01:15:23.73", "status": 0}};
```

Now, your flow should look like this:

![nora3](https://i.imgur.com/D83bC8L.png)

Now you just need to add the device to your Google Home. Open the Google Home app on your phone and follow the these steps:

![nora4](https://i.imgur.com/8OvWX1X.jpg)

You can now assign the Sensor Hub to any room. Try asking Google Assistant "Ok Google, what is the temperature inside?" to get the current temperature from the Hub. You can also ask "Ok Google, set the temperature to 21 degrees" to publish the setpoint over MQTT.

### <a name='AmazonAlexa'></a>Amazon Alexa

For Amazon Alexa, we will be adding temperature and motion sensors to our smart home using the Alexa nodes that we added earlier.

First, create an account [here](https://red.cb-net.co.uk/new-user) and create both a temperature sensor and a motion sensor [here](https://red.cb-net.co.uk/devices).

![alexa1](https://i.imgur.com/zwPTQ5X.png)

Add two `alexa smart home v3 state` nodes, one for each sensor. Double-click on them and add your account. You should then see your devices populate here:

![alexa2](https://i.imgur.com/58gnEQA.png)

With the nodes selected, you can view their acceptable input in the Help sidebar:

![alexa3](https://i.imgur.com/YouyGeO.png)

Create `mqtt in` nodes for temperature (topic: `events/object/rawTemperature`) and occupancy (topic: `events/object/combinedOccupancy`).

At this point, your flow should look like this:

![alexa4](https://i.imgur.com/xjpxkxo.png)

As you saw previously, the `mqtt in` node outputs something like:

```json
{
  "Present_Value": 25.32,
  "Units": "°C",
  "updated": "31-07-2020 02:18:43.0",
  "status": 0
}
```

But the temperature sensor would expect this as its input:

```json
{
    "state": {
        "temperature": 25.32
    }
}
```

So we can add a `function` node between the `mqtt in` and temperature sensor nodes to handle this. The code is as follows:

```javascript
var jsonPayload = JSON.parse(msg.payload);
var temperature = Number(jsonPayload.Present_Value);

return { "acknowledge" : true, "payload" : { "state" : { "temperature" : temperature } } };
```

The motion sensor is a bit different though as it is doesn't give the number of occupants, but only whether it is occupied or not instead.

We can use a `function` node between the `mqtt in` node and motion sensor with the following code to deal with this:

```javascript
var jsonPayload = JSON.parse(msg.payload);
var combinedOccupancy = jsonPayload.Present_Value;

if (combinedOccupancy === 0) {
    return { "acknowledge" : true, "payload" : { "state" : { "motion" : "NOT_DETECTED" } } };
} else {
    return { "acknowledge" : true, "payload" : { "state" : { "motion" : "DETECTED" } } };
}
```

Now, your flow should look like this:

![alexa5](https://i.imgur.com/z3Nhj8J.png)

Now you just need to add the device to Amazon Alexa. Open the Alexa app on your phone and follow the these steps:

![alexa6](https://i.imgur.com/4q6KQjo.jpg)

You can now assign the Sensor Hub to any room/group. Try asking "Alexa, what is the temperature inside?" to get the current temperature from the Hub. You can also try setting up routines using the motion sensor and other smart devices. In the following example, I created a routine to turn off the light if no motion has been detected in the last 30 minutes during the day on weekdays:

<img src="https://i.imgur.com/4GJNzbO.jpg" width="300">

## <a name='End'></a>End

[Here](https://drive.google.com/file/d/1HDF5eD8VtYeA42fMrsJmodwcQ2KnExV7/view?usp=sharing) is the complete flow for this tutorial (you will have to configure the internal broker login, nora token, and alexa account for it to work correctly).

### <a name='Resources'></a>Resources

- [Node-RED Documentation](https://nodered.org/docs/)
- [Node-RED MQTT Recipes](https://cookbook.nodered.org/mqtt/)
- [node-red-contrib-nora](https://flows.nodered.org/node/node-red-contrib-nora)
- [node-red-contrib-alexa-smart-home](https://flows.nodered.org/node/node-red-contrib-alexa-smart-home)
