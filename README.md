# Sparkplug B Tutorial: Pico W LED Control and Temperature Monitoring

## Introduction to Sparkplug B

Sparkplug B is a specification for MQTT-based communication in Industrial Internet of Things (IIoT) scenarios. It provides a standardized way to structure MQTT topics and payloads, enhancing interoperability in IIoT systems. Key features include:

- Defined topic namespace: Sparkplug B specifies a consistent topic structure, ensuring that data from different devices and vendors can be easily understood and processed by the system.
- Standardized payload format: It uses a unified data format, making it easier to integrate devices from different manufacturers.
- Birth and death certificates for nodes and devices: Sparkplug B tracks device lifecycle events, which improves system reliability by knowing when a device is online (birth) or offline (death).
- State management: It helps maintain the state of devices and nodes, which is crucial for monitoring and controlling IIoT systems in real-time.

### Why Use Sparkplug B?

In an automated laboratory, different devices such as robots, sensors, pumps, and controllers often need to communicate with a central orchestrator for coordination and data sharing. Traditional communication protocols may lack standardization in data structure, making it difficult to integrate various devices seamlessly.

Sparkplug B solves these challenges by providing a standardized approach for MQTT communication, which brings several benefits to an automated lab:

- Improved Interoperability

When multiple instruments from different manufacturers are used in the lab (e.g., temperature sensors, robotic arms, and liquid handling robots), Sparkplug B ensures that data can be exchanged in a common format. This eliminates the need for custom integration for each device.
Enhanced Device Monitoring and Control

For example, in an automated lab, you may have different devices monitoring environmental conditions, such as temperature, humidity, or pH levels. With Sparkplug B, you can easily monitor the state of each device and receive alerts if a device goes offline (using the birth and death certificate functionality).
This is crucial for maintaining lab safety and ensuring that critical experiments are not disrupted due to device failures.

- Standardized State Management

Sparkplug B's state management allows the system to keep track of each device's current status. For instance, in a chemical synthesis process, a robotic arm may need to know the exact state of a temperature sensor before proceeding with the next step. Sparkplug B ensures that the real-time state information is available and standardized.
Automated Workflows and Coordination

In a self-driving lab, tasks such as mixing chemicals, adjusting temperatures, or collecting samples can be coordinated more efficiently using Sparkplug B. For example, a robotic arm can send an MQTT message structured according to Sparkplug B to adjust the settings of a temperature controller based on feedback from a temperature sensor.
The standardized communication reduces the risk of misinterpretation between devices, allowing for more reliable and automated workflows.

In this tutorial, we'll use Sparkplug B to control the onboard LED and monitor the temperature sensor of a Raspberry Pi Pico W, demonstrating how Sparkplug B can standardize communication and improve device interoperability in an automated laboratory setup.

## Hardware Setup

We'll use the same Pico W setup as in the previous tutorial:

- Raspberry Pi Pico W
- Onboard LED
- Onboard temperature sensor

Ensure your Pico W is set up and connected to WiFi as described in the previous tutorial.

## Software Setup

On your Pico W, you'll need:

1. MicroPython firmware
2. `mqtt_as.py` module
3. `sparkplug_b.py` module (a MicroPython-compatible version)

On your computer, install the following Python packages:

```bash
pip install paho-mqtt sparkplug-b matplotlib
```

You'll also need an MQTT broker. We'll use HiveMQ Cloud for this tutorial, but you can use any MQTT broker that supports Sparkplug B.

## Pico W Script (sparkplug_pico.py)

Create a new file on your Pico W called `sparkplug_pico.py`:

```python
import time
from machine import Pin, ADC
from mqtt_as import MQTTClient
import network
import sparkplug_b as spb

# WiFi and MQTT settings
SSID = "Your_SSID"
WiFi_PASS = "Your_WiFi_Password"
MQTT_BROKER = "Your_HiveMQ_Broker_Address"
MQTT_USER = "Your_HiveMQ_Username"
MQTT_PASS = "Your_HiveMQ_Password"

# Sparkplug B settings
GROUP_ID = "Pico_W_Group"
EDGE_NODE_ID = "Pico_W_Node"
DEVICE_ID = "Pico_W_Device"

# Hardware setup
led = Pin("LED", Pin.OUT)
sensor_temp = ADC(4)

# Connect to WiFi
def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('Connecting to WiFi...')
        wlan.connect(SSID, WiFi_PASS)
        while not wlan.isconnected():
            pass
    print('WiFi connected')

# Read temperature
def read_temp():
    adc = sensor_temp.read_u16()
    volt = (3.3 / 65535) * adc
    temp = 27 - (volt - 0.706) / 0.001721
    return round(temp, 1)

# MQTT callbacks
def on_message(topic, msg, retained):
    try:
        payload = spb.decodePayload(msg)
        for metric in payload.metrics:
            if metric.name == "LED_Command":
                if metric.value:
                    led.on()
                    print("LED ON")
                else:
                    led.off()
                    print("LED OFF")
    except Exception as e:
        print(f"Error processing message: {e}")

# Main function
async def main():
    connect_wifi()
    
    client = MQTTClient(MQTT_BROKER, user=MQTT_USER, password=MQTT_PASS)
    await client.connect()
    
    while True:
        temp = read_temp()
        payload = spb.encodePayload({
            "Temperature": spb.FloatMetric(temp)
        })
        await client.publish(f"spBv1.0/{GROUP_ID}/DDATA/{EDGE_NODE_ID}/{DEVICE_ID}", payload)
        print(f"Published temperature: {temp}°C")
        await asyncio.sleep(5)

# Run the main function
import asyncio
asyncio.run(main())
```

## Computer Script (sparkplug_control.py)

On your computer, create a script to control the LED and monitor the temperature:

```python
import time
import paho.mqtt.client as mqtt
from sparkplug_b import *
import matplotlib.pyplot as plt
from IPython.display import clear_output

# MQTT and Sparkplug B settings
MQTT_BROKER = "Your_HiveMQ_Broker_Address"
MQTT_PORT = 8883  # HiveMQ Cloud uses port 8883 for secure connections
MQTT_USER = "Your_HiveMQ_Username"
MQTT_PASS = "Your_HiveMQ_Password"
GROUP_ID = "Pico_W_Group"
EDGE_NODE_ID = "Pico_W_Node"
DEVICE_ID = "Pico_W_Device"

temperatures = []
timestamps = []

def on_connect(client, userdata, flags, rc):
    print(f"Connected with result code {rc}")
    client.subscribe(f"spBv1.0/{GROUP_ID}/DDATA/{EDGE_NODE_ID}/{DEVICE_ID}")

def on_message(client, userdata, msg):
    try:
        payload = sparkplug_b_pb2.Payload()
        payload.ParseFromString(msg.payload)
        for metric in payload.metrics:
            if metric.name == "Temperature":
                temp = metric.float_value
                temperatures.append(temp)
                timestamps.append(time.time())
                print(f"Received temperature: {temp:.1f}°C")
                
                # Update plot
                clear_output(wait=True)
                plt.figure(figsize=(10, 6))
                plt.plot(timestamps, temperatures)
                plt.title("Pico W Temperature over Time")
                plt.xlabel("Time")
                plt.ylabel("Temperature (°C)")
                plt.show()
    except Exception as e:
        print(f"Error processing message: {e}")

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message
client.username_pw_set(MQTT_USER, MQTT_PASS)
client.tls_set()  # Enable SSL/TLS
client.connect(MQTT_BROKER, MQTT_PORT, 60)

client.loop_start()

def send_led_command(state):
    payload = sparkplug_b_pb2.Payload()
    add_metric_to_payload(payload, "LED_Command", None, MetricDataType.Boolean, state)
    client.publish(f"spBv1.0/{GROUP_ID}/NCMD/{EDGE_NODE_ID}", payload.SerializeToString(), 0, False)
    print(f"Sent LED {'ON' if state else 'OFF'} command")

try:
    while True:
        send_led_command(True)
        time.sleep(5)
        send_led_command(False)
        time.sleep(5)
except KeyboardInterrupt:
    print("Exiting...")
    client.loop_stop()
    client.disconnect()
```

## Running the Example

1. Upload the `sparkplug_pico.py` script to your Pico W and run it.
2. Run the `sparkplug_control.py` script on your computer.

You should see the Pico W's onboard LED toggling every 5 seconds, and a real-time plot of the temperature data on your computer.

## Understanding Sparkplug B in this Example

1. **Topic Namespace**: We use the Sparkplug B topic structure (e.g., `spBv1.0/Pico_W_Group/DDATA/Pico_W_Node/Pico_W_Device`).
2. **Payload Format**: We use the Sparkplug B payload format to structure our data, including specific metric types.
3. **Interoperability**: By following the Sparkplug B specification, our Pico W could potentially work with other Sparkplug B-compliant systems.
4. **Efficiency**: Sparkplug B uses Protocol Buffers for serialization, which is more efficient than plain text formats.

This example demonstrates how Sparkplug B can be used in a real IoT scenario with actual hardware. It provides a standardized way to handle device communication, which becomes crucial when scaling up to larger systems with multiple devices and applications.

# Advanced Sparkplug B Tutorial: Camera Monitoring for Experimental Equipment

## Introduction

In this advanced tutorial, we'll demonstrate how to use Sparkplug B to monitor data streams from a camera observing experimental equipment. We'll implement real-time anomaly detection and trigger alerts or responses when abnormal data is detected.

## System Overview

1. A camera continuously monitors experimental equipment.
2. Real-time analysis of the video stream detects anomalies.
3. When an anomaly is detected, the system:
   - Sends an alert using Sparkplug B
   - Logs the event to MongoDB
   - Triggers a Prefect workflow for further processing

## Hardware Setup

- Raspberry Pi 4 (or similar) with camera module
- (Optional) Relay module for controlling external devices

## Software Requirements

Install the following Python packages:

```bash
pip install paho-mqtt sparkplug-b opencv-python pymongo prefect
```

## Camera Monitoring Script (camera_monitor.py)


## Control System Script (control_system.py)


## Running the Example

1. Start the `camera_monitor.py` script on the Raspberry Pi with the camera.
2. Run the `control_system.py` script on your main computer or server.

## System Workflow

1. The camera continuously monitors the experimental equipment.
2. When an anomaly is detected (in this example, unusual brightness levels):
   - An alert is sent using Sparkplug B.
   - The control system receives the alert and:
     - Logs the alert to MongoDB
     - Triggers a Prefect workflow for notification and further processing

## Sparkplug B Application in this Example

1. **Standardized Topic Structure**: We use Sparkplug B's topic structure for alerts (DALERT).
2. **Flexible Payload Format**: We use Sparkplug B payload to transmit anomaly alert messages.
3. **Device State Management**: Although not shown in this simple example, Sparkplug B can be used to track the camera's state (active, fault, etc.).
4. **Birth and Death Certificates**: Sparkplug B's birth and death certificates can be implemented to monitor the camera's lifecycle and trigger reconnection attempts if it goes offline.

## Conclusion

This advanced example demonstrates how to use the Sparkplug B protocol in a laboratory setting for camera-based monitoring of experimental equipment. By integrating real-time video analysis, anomaly detection, and automated alert systems, we've created a system that can enhance laboratory safety and experiment monitoring. This approach showcases the flexibility of Sparkplug B in handling various types of data and events in an IoT ecosystem.

## Further Enhancements

- **Enhance Anomaly Detection Logic**:
  - Implement more sophisticated detection methods: The current anomaly detection is based solely on average brightness, which is simplistic and may cause false positives. Consider using more advanced computer vision techniques (e.g., image segmentation, feature detection) or machine learning models (e.g., deep learning) to improve detection accuracy.
  - Adaptive thresholding: Instead of using fixed brightness thresholds (50 and 200), adjust thresholds dynamically based on experimental conditions. This could involve calculating thresholds from historical data.

- **Improve Anomaly Handling**:
  - Log anomaly events: Save frames where anomalies are detected locally or upload them to cloud storage for future analysis.
  - Multi-level alerts: Configure different alert levels based on the severity of anomalies. For instance, log minor anomalies and send emergency alerts for severe cases.

- **Optimize MQTT and Sparkplug B Integration**:
  - Set QoS: In `client.publish()`, configure the Quality of Service (QoS) to ensure reliable message delivery. Consider setting QoS to 1 (at least once) or 2 (exactly once).
  - Auto-reconnect and disconnection handling: Implement automatic reconnection when the connection to the MQTT broker is lost to ensure system reliability.

- **Add Logging Functionality**:
  - Log records: Use Python's `logging` module to log debug information and exception events, which will facilitate troubleshooting.
  - Log levels: Utilize different logging levels (e.g., `INFO`, `WARNING`, `ERROR`) to enhance the readability of log messages.

- **Optimize MongoDB and Notification Operations**:
  - Batch MongoDB operations: If the frequency of messages is high, consider batch writing multiple alerts to MongoDB to reduce frequent database operations.
  - Asynchronous notification: Use asynchronous libraries (e.g., `asyncio`) or multithreading to execute `send_notification()` without blocking the main thread.

These key improvements can enhance the system's functionality, reliability, and performance.

