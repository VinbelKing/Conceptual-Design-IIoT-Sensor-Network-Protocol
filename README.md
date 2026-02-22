Conceptual-Design-IIoT-Sensor-Network-Protocol
Python simulation of an IIoT sensor network generating and visualizing real-time data via MQTT, CoAP, and OPC UA.

1. Create project and virtual environment
Open PowerShell and run these commands:

powershell
# 1. Create project folder, then cd into it
mkdir iiot_simulation
cd iiot_simulation

# 2. Create virtual environment
python -m venv venv

# 3. Activate venv (Windows)
.\venv\Scripts\activate
On Windows 11, .\venv\Scripts\activate is the correct activation command in PowerShell.
​

2. Install Python packages and Mosquitto
With the venv active:

powershell
pip install pandas numpy paho-mqtt aiocoap asyncua matplotlib
These packages provide MQTT (paho-mqtt), CoAP (aiocoap), OPC UA (asyncua), and plotting (matplotlib, pandas, numpy).

Install Mosquitto (MQTT broker) on Windows 11:

powershell
winget install -e --id EclipseFoundation.Mosquitto
If winget is not available, download the Windows x64 installer (mosquitto-*-install-windows-x64.exe) from the official site and run it, which installs into C:\Program Files\mosquitto.

If mosquitto is “not recognized”, start it from the install folder:

powershell
cd "C:\Program Files\mosquitto"
.\mosquitto.exe -v
3. Create the three sensor simulation scripts
All files go into iiot_simulation/.

3.1 mqtt_sensor_simulation.py
python
import paho.mqtt.client as mqtt
import random
import time

broker = "localhost"
port = 1883
topic = "sensor/data"

def simulate_sensor_data():
    while True:
        temperature = random.uniform(20.0, 25.0)
        humidity = random.uniform(30.0, 50.0)
        payload = f'{{"temperature": {temperature}, "humidity": {humidity}}}'
        client.publish(topic, payload)
        print("MQTT published:", payload)
        time.sleep(1)

client = mqtt.Client()
client.connect(broker, port)
simulate_sensor_data()
This is a simple MQTT publisher loop using paho-mqtt, publishing a JSON payload every second.

3.2 coap_sensor_simulation.py
You need a CoAP server running at coap://localhost/sensor/data for this client to succeed; for the assignment, the client code itself is:

python
import asyncio
import random
from aiocoap import *

async def simulate_sensor_data():
    protocol = await Context.create_client_context()
    while True:
        temperature = random.uniform(20.0, 25.0)
        humidity = random.uniform(30.0, 50.0)
        payload = f'{{"temperature": {temperature}, "humidity": {humidity}}}'.encode("utf-8")
        request = Message(code=POST, payload=payload)
        request.set_request_uri("coap://localhost/sensor/data")
        try:
            response = await protocol.request(request).response
            print("CoAP result:", response.code, response.payload)
        except Exception as e:
            print("CoAP error:", e)
        await asyncio.sleep(1)

if __name__ == "__main__":
    asyncio.run(simulate_sensor_data())
If you don’t have a CoAP server, you can still show this as the client‑side simulation; in a full setup you’d also implement a minimal CoAP resource at /sensor/data.

3.3 opcua_sensor_simulation.py
python
from asyncua import ua, Server
import asyncio
import random

async def main():
    server = Server()
    await server.init()
    server.set_endpoint("opc.tcp://0.0.0.0:4840/freeopcua/server/")
    uri = "http://examples.freeopcua.github.io"
    idx = await server.register_namespace(uri)

    objects = await server.nodes.objects
    myobj = await objects.add_object(idx, "MyObject")
    temperature = await myobj.add_variable(idx, "Temperature", 0.0)
    humidity = await myobj.add_variable(idx, "Humidity", 0.0)
    await temperature.set_writable()
    await humidity.set_writable()

    print("OPC UA server started at opc.tcp://0.0.0.0:4840/freeopcua/server/")

    async with server:
        while True:
            temp_value = random.uniform(20.0, 25.0)
            hum_value = random.uniform(30.0, 50.0)
            await temperature.write_value(temp_value)
            await humidity.write_value(hum_value)
            print(f"OPC UA values - Temperature: {temp_value:.2f}, Humidity: {hum_value:.2f}")
            await asyncio.sleep(1)

if __name__ == "__main__":
    asyncio.run(main())
This follows the standard asyncua pattern: init server, register namespace, add variables, then update them in a loop.

4. Real‑time MQTT data visualization
Create data_visualization.py:

python
import paho.mqtt.client as mqtt
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime
import ast

data = []

def on_message(client, userdata, message):
    payload = message.payload.decode("utf-8")
    data.append((datetime.now(), payload))
    if len(data) > 100:
        data.pop(0)

    df = pd.DataFrame(data, columns=["timestamp", "sensor_data"])
    df["temperature"] = df["sensor_data"].apply(lambda x: ast.literal_eval(x)["temperature"])
    df["humidity"] = df["sensor_data"].apply(lambda x: ast.literal_eval(x)["humidity"])

    plt.clf()
    plt.plot(df["timestamp"], df["temperature"], label="Temperature (°C)")
    plt.plot(df["timestamp"], df["humidity"], label="Humidity (%)")
    plt.xlabel("Time")
    plt.ylabel("Value")
    plt.legend()
    plt.gcf().autofmt_xdate()
    plt.draw()
    plt.pause(0.1)

client = mqtt.Client()
client.connect("localhost", 1883)
client.subscribe("sensor/data")
client.on_message = on_message

plt.ion()
plt.figure()
client.loop_start()
plt.show()

# Keep the script alive
while True:
    plt.pause(1.0)
Using ast.literal_eval is safer than eval when parsing simple JSON‑like strings.

You can later adapt this pattern to CoAP and OPC UA by replacing the MQTT callback with data collection from those protocols and feeding the same plotting logic.

5. Running everything (Windows 11)
You’ll use multiple terminals (all from inside the iiot_simulation folder with venv activated, except maybe the Mosquitto one).

5.1 Start Mosquitto broker
Terminal 1 (PowerShell):

powershell
cd "C:\Program Files\mosquitto"
.\mosquitto.exe -v
Or, if mosquitto is on PATH:

powershell
mosquitto -v
5.2 Run sensor simulations
Open three more terminals, activate the venv, cd into iiot_simulation, then run:

Terminal 2 – MQTT:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python mqtt_sensor_simulation.py
Terminal 3 – CoAP (client simulation):

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python coap_sensor_simulation.py
Terminal 4 – OPC UA server:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python opcua_sensor_simulation.py
5.3 Run visualization
In VS Code’s integrated terminal or another PowerShell window:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python data_visualization.py
A Matplotlib window should show two lines (temperature and humidity) updating in near real time as MQTT messages arrive.
​

You can take screenshots (PNG) and a short screen recording (MP4) of the visualization for your visualizations/ folder.

6. Project structure and GitHub
Organize your folder like this (create empty placeholders where needed):

text
iiot_simulation/
├── README.md
├── mqtt_sensor_simulation.py
├── coap_sensor_simulation.py
├── opcua_sensor_simulation.py
├── data_visualization.py
├── visualizations/
│   ├── mqtt_visualization.png
│   ├── coap_visualization.png
│   ├── opcua_visualization.png
│   └── visualization_demo.mp4
└── comparison_report.pdf
Use GitHub Desktop:

Open GitHub Desktop → File → New Repository.

Name: iiot_simulation, set local path to this folder, click Create Repository.

You’ll see all files as changes; enter a message like Initial commit for IIoT sensor network simulation and click Commit to main.

Click Publish repository / Push origin to upload to GitHub.
​

If you want, I can next help you:

Write a concise README.md explaining how to run each component, or

Sketch the comparison points for MQTT vs CoAP vs OPC UA for your comparison_report.pdf.

yes, Help: Write a concise README.md explaining how to run each component, or
Sketch the comparison points for MQTT vs CoAP vs OPC UA for your comparison_report.pdf.
Below is a concise README.md plus a bullet list you can reuse in comparison_report.pdf.

README.md (put this in your project root)
text
# IIoT Sensor Network Simulation

This project simulates an Industrial Internet of Things (IIoT) sensor network using three protocols: **MQTT**, **CoAP**, and **OPC UA**. It generates random temperature and humidity readings and visualizes MQTT sensor data in real time.

## 1. Prerequisites

- Python 3.10+  
- Windows 11 64-bit (tested)  
- Mosquitto MQTT broker

## 2. Setup

### 2.1. Create project and virtual environment

```powershell
mkdir iiot_simulation
cd iiot_simulation

python -m venv venv
.\venv\Scripts\activate
2.2. Install Python dependencies
powershell
pip install pandas numpy paho-mqtt aiocoap asyncua matplotlib
2.3. Install Mosquitto (MQTT broker)
On Windows 11, either:

Use the package manager:

powershell
winget install -e --id EclipseFoundation.Mosquitto
Or download the 64-bit Windows installer from the official Mosquitto site, run it, and accept defaults (typically installs to C:\Program Files\mosquitto).

3. Running the System
You will use multiple terminals. In each Python terminal, first:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
3.1. Start the MQTT broker
Terminal 1:

powershell
cd "C:\Program Files\mosquitto"
.\mosquitto.exe -v
(If mosquitto is on PATH, you can simply run mosquitto -v.)

3.2. Run MQTT sensor simulation
Terminal 2:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python mqtt_sensor_simulation.py
This script publishes random temperature and humidity JSON messages every second on the topic sensor/data.

3.3. Run CoAP sensor simulation
Terminal 3:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python coap_sensor_simulation.py
This script acts as a CoAP client, sending POST requests with simulated sensor values to coap://localhost/sensor/data. A CoAP server endpoint at this URI is assumed for full end-to-end behavior.

3.4. Run OPC UA sensor simulation
Terminal 4:

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python opcua_sensor_simulation.py
This script starts an OPC UA server at opc.tcp://0.0.0.0:4840/freeopcua/server/ and continuously updates the Temperature and Humidity variables under the MyObject node.

3.5. Run real-time MQTT data visualization
Terminal 5 (or the integrated terminal in your IDE):

powershell
cd path\to\iiot_simulation
.\venv\Scripts\activate
python data_visualization.py
A Matplotlib window opens and displays live plots of temperature and humidity over time for the MQTT sensor data (latest ~100 samples).

You can capture screenshots and short recordings of the plots and place them in visualizations/.

4. Project Structure
text
iiot_simulation/
├── README.md
├── mqtt_sensor_simulation.py
├── coap_sensor_simulation.py
├── opcua_sensor_simulation.py
├── data_visualization.py
├── visualizations/
│   ├── mqtt_visualization.png
│   ├── coap_visualization.png
│   ├── opcua_visualization.png
│   └── visualization_demo.mp4
└── comparison_report.pdf
5. Notes
All scripts assume services are running on localhost using default ports (MQTT: 1883, OPC UA: 4840).

CoAP support requires a compatible CoAP server if you want to see responses.

The provided visualization focuses on MQTT; similar logic can be adapted for CoAP and OPC UA.

text

***

## Comparison points for `comparison_report.pdf`

Use or adapt these bullets and turn them into paragraphs or a table:

### MQTT

- **Protocol type**: Application-layer publish/subscribe protocol over TCP.  
- **Communication model**: Broker-based, topics and subscriptions; decouples producers and consumers.  
- **Overhead and footprint**: Very lightweight header, small code size; designed for constrained devices and unreliable networks.  
- **Quality of Service**: QoS 0, 1, 2 provide “at most once”, “at least once”, and “exactly once” delivery levels.  
- **Typical use cases**: Telemetry, IIoT sensor data collection, mobile/edge devices, cloud ingestion.  
- **Strengths in this project**: Simple to implement, easy real-time visualization, natural fit for streaming sensor values.  
- **Limitations**: No built-in data modeling or rich semantics; relies on custom payload formats and broker configuration.

### CoAP

- **Protocol type**: RESTful, UDP-based protocol inspired by HTTP for constrained devices.  
- **Communication model**: Client–server with methods similar to HTTP (GET, POST, PUT, DELETE) and URI-based resources.  
- **Overhead and footprint**: Very low overhead, supports confirmable and non-confirmable messages, good for lossy networks.  
- **Reliability**: Built-in retransmission for confirmable messages, but still less reliable than TCP-based protocols in some scenarios.  
- **Typical use cases**: Resource-oriented access to sensors/actuators, constrained networks (LoRaWAN, 6LoWPAN, etc.).  
- **Strengths in this project**: Natural fit for modeling sensors as REST resources; simple request/response interactions.  
- **Limitations**: Requires CoAP-specific infrastructure; less common tooling on desktops; not as plug-and-play for streaming visualization as MQTT.

### OPC UA

- **Protocol type**: Industrial interoperability standard with rich information modeling, typically over TCP.  
- **Communication model**: Client–server; supports browsing, subscriptions, method calls, and complex data types.  
- **Data modeling**: Strongly typed address space with objects, variables, methods, and hierarchical namespaces.  
- **Security**: Built-in security model (encryption, signing, authentication, certificates) suitable for industrial environments.  
- **Typical use cases**: Industrial automation, integration of PLCs, SCADA, MES, and higher-level systems.  
- **Strengths in this project**: Realistic representation of industrial assets (e.g., `MyObject.Temperature` and `MyObject.Humidity`), aligns with factory/plant architectures.  
- **Limitations**: Heavier protocol and implementation; steeper learning curve; more complex to set up than MQTT or CoAP.

### Overall comparison for IIoT sensor networks

- **Scalability**: MQTT scales well for many sensors and subscribers; OPC UA scales in richer industrial systems but with more complexity; CoAP scales in constrained networks with simple resources.  
- **Latency and overhead**: CoAP and MQTT are optimized for low overhead; OPC UA trades higher overhead for richer semantics and services.  
- **Ease of integration**: MQTT integrates easily with cloud platforms and dashboards; OPC UA integrates natively with industrial tooling; CoAP integration depends more on specialized gateways.  
- **Best fit in this assignment**:  
  - MQTT: primary channel for streaming sensor data and visualization.  
  - CoAP: example of REST-style interaction with constrained devices.  
  - OPC UA: example of industrial-grade, semantically rich sensor exposure.
