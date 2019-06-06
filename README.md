# Azure Digital Twins with the Microsoft MXchip AZ3166

## Goal

The goal of this project is to connect the [Microsoft MX chip](https://microsoft.github.io/azure-iot-developer-kit/) directly to the [Microsoft Azure Digital Twins](https://azure.microsoft.com/en-us/services/digital-twins/). The current solutions all use the IoT hub as entry point. In Azure Digital Twins the IoT Hub is also part of the solution, but there are extra message properties needed for accepting the messages from the MXchip. Because debugging of messages are hard currently in Azure Digital Twins, this project has created.

![High Level Solution](https://github.com/rploeg/Azure-Digital-Twins-and-MXchip/blob/master/highlevelsolution.png)

## Setup Azure Digital Twins

First you need to setup an Azure Digital Twins instance with a graph. The Microsoft team has created a perfect guideline how you need to create a graph like this. Please check [this](https://docs.microsoft.com/en-us/azure/digital-twins/tutorial-facilities-setup) link. 

There are some important things you don't need to forgot: 

* Create a simple graph structure (tenant, building, floor, room(this is the space where the sensors are connected)
* Create a minimal structure of one device with two sensors in the graph that are connected to a space
* Create two matcher, one for temperature and one for humidity
* Connected the UDF's to the right space
* Give the right role assignment to the UDF's

### Matcher

First of all you need to create matchers for the two sensor types:

* Get all humidity sensors

```javascript
{
  "SpaceId": "YOUR_SPACE_ID",
  "Name": "Humidity matcher",
  "Description": "All humidity sensors",
  "Conditions": [
    {
      "target": "Sensor",
      "path": "$.dataType",
      "value": "\"Humidity\"",
      "comparison": "Equals"
    }
  ]
}
```

* get all temperature sensors

```javascript
{
  "SpaceId": "YOUR_SPACE_ID",
  "Name": "Temperature matcher",
  "Description": "All temperature sensors",
  "Conditions": [
    {
      "target": "Sensor",
      "path": "$.dataType",
      "value": "\"Temperature\"",
      "comparison": "Equals"
    }
  ]
}
```

So when temperature or humidity data gets now in the Azure Digital Twins, you get them.

### UDF - set value on space

When we get with the matcher the right sensors, we can create and UDF. UDF is a javascript based rule engine within Azure Digital Twins. In this example I only set a temperature value on a space (the room)

```javascript
function process(telemetry, executionContext) {
    var message = JSON.parse(telemetry.SensorReading);
    const sensorMeta = getSensorMetadata(telemetry.SensorId);
    const sensorValue = message.SensorValue;
    const space = sensorMeta.Space();
    setSpaceValue(sensorMeta.SpaceId, "Temperature", sensorValue);
    catch (err) {
        log(`Error in updating space value. Details: ${JSON.stringify(err)}`);
    }
}
```

### UDF - if humidity is above 50

But we want to use also some rules in the javascript. For example when the humidity is higher then 50% the humidity space property need to be updated with that the humidity is too high in the room. We then can send a notification to the egress point of Azure Digital Twins, so we can create an action.

```javascript
function process(telemetry, executionContext) {
    var message = JSON.parse(telemetry.SensorReading);
    const sensorMeta = getSensorMetadata(telemetry.SensorId);
    const sensorValue = message.SensorValue;
    sendNotification(sensorMeta.SpaceId, "Space", JSON.stringify(message));
    const space = sensorMeta.Space();
    setSpaceValue(sensorMeta.SpaceId, "HumidityLevel", sensorValue);
    try {
        if (sensorValue > "50") {
            setSpaceValue(sensorMeta.SpaceId, "HumidityLevel", "High");
            sendNotification(sensorMeta.SpaceId, "Space", `Too high humidity: ${sensorValue}`);
        }
        else if (sensorValue < "50") {
            setSpaceValue(sensorMeta.SpaceId, "HumidityLevel", "Low");
        }
        else {
            log(`Invalid sensor reading was found, no action was taken: ${sensorValue}`);
            setSpaceValue(sensorMeta.SpaceId, "HumidityLevel", "ERROR");
        }
    }
    catch (err) {
    sendNotification(sensorMeta.SpaceId, "Space", `Invalid sensor reading was found, no action was taken: ${sensorValue}`);
        log(`Error in updating space value. Details: ${JSON.stringify(err)}`);
    }
}
```

Don't forget to create the two role assignments on this UDF, else they won't be executed. 

## Configure MXChip

So now Azure Digital Twins is configured, we can focus us on the MXchip. I have used the [get started project](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-arduino-iot-devkit-az3166-get-started) code of Microsoft itself. I have changed some code, the most important one is the following (see the repo for the full project):

```
      char messagePayload[MESSAGE_MAX_LEN];
      bool temperatureAlert = readTempMessage(messagePayload, &temperature);
      EVENT_INSTANCE* message = DevKitMQTTClient_Event_Generate(messagePayload, MESSAGE);
      DevKitMQTTClient_Event_AddProp(message, "DigitalTwins-Telemetry", "1.0");
      DevKitMQTTClient_Event_AddProp(message, "DigitalTwins-SensorHardwareId", "SENSORHARDWAREID");
      DevKitMQTTClient_Event_AddProp(message, "CreationTimeUtc", "2019-06-03T13:45:30.0000000Z");
      DevKitMQTTClient_SendEventInstance(message);
```

To send messages to Azure Digital Twins you need to add the right message properties to the message, else it will not routed within the platform. 

## Result

If we now start the MXchip every 5 seconds a a new message will be sended, it will send now the the right message format and the corresponding space will be updated with Azure Digital Twins. You can check that with [Postman](https://www.postman) via the following URL on Azure Digital Twins:

```
{{baseurl}}spaces/SPACEID?includes=values
```
Then you will get the values of the corresponding space, the result should something like this:

```javascript
{
    "values": [
        {
            "type": "HumidityLevel",
            "value": "High",
            "timestamp": "2019-06-05T19:19:24.3039286Z",
            "historicalValues": [
                {
                    "value": "High"
                },
                {
                    "value": "High"
                }
            ]
        },
        {
            "type": "Temperature",
            "value": "25.10",
            "timestamp": "2019-06-05T19:19:23.4640624Z",
            "historicalValues": [
                {
                    "value": "25.10"
                },
                {
                    "value": "25.10"
                },
                {
                    "value": "25.10"
                }
            ]
        },
        {
            "type": "Humidity",
            "value": "56.30",
            "timestamp": "2019-06-05T19:16:25.20284Z",
            "historicalValues": [
                {
                    "value": "56.30"
                },
                {
                    "value": "56.20"
                },
                {
                    "value": "56.30"
                }
            ]
        },
        {
            "type": "CO2",
            "value": "971",
            "timestamp": "2019-06-05T11:26:09.4384065Z",
            "historicalValues": [
                {
                    "value": "971"
                },
                {
                    "value": "929"
                }
            ]
        }
    ],
    "id": "1ce8b0d9-96a8-47a7-a531-82ecbc7303b2",
    "name": "Room01",
    "friendlyName": "Room01 for testing MXchip",
    "typeId": 64,
    "parentSpaceId": "6be8cf5c-44c6-46c6-b626-041faee261bc",
    "subtypeId": 13,
    "statusId": 12
}
```

