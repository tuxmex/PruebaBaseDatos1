### Tutorial: Control de Ventilador con ESP32, DHT22, Node-RED y PostgreSQL

Este tutorial te guiará en la creación de un sistema de control de ventilador usando un ESP32, un sensor DHT22, Node-RED para el procesamiento de datos y PostgreSQL para almacenar las lecturas de temperatura. Además, se simulará un ventilador mediante un LED conectado al ESP32 en Wokwi.

#### **1. Simulación en Wokwi**
Comenzaremos configurando el circuito en Wokwi. Asegúrate de tener los siguientes componentes:

- **ESP32**
- **DHT22 Sensor**
- **LED (para simular el ventilador)**
- **Relay (para controlar el LED)**

#### **Paso 1: Configuración del Circuito**

1. **Conecta el DHT22 al ESP32:**
   - **VCC** a **3.3V** en el ESP32.
   - **GND** a **GND** en el ESP32.
   - **DATA** a **GPIO15** en el ESP32 (puedes usar cualquier pin GPIO disponible).

2. **Conecta el Relay al LED y al ESP32:**
   - Conecta el **VCC** del relay a **3.3V** del ESP32.
   - Conecta **GND** del relay a **GND** del ESP32.
   - Conecta el pin de **señal** del relay al pin **GPIO2** del ESP32.
   - Conecta el LED en serie con el relay. Una pierna del LED al pin NO (Normally Open) del relay y la otra al GND del ESP32, asegurándote de usar una resistencia adecuada.

Aquí está el enlace al proyecto Wokwi: [Wokwi Simulation](https://wokwi.com/projects/406233213917645825).

#### **Paso 2: Código MicroPython en ESP32**

Copia y pega el siguiente código en el editor de MicroPython de Wokwi:

```python
import machine
import time
import dht
import ubinascii
from umqtt.simple import MQTTClient

# Configuración del DHT22
dht_sensor = dht.DHT22(machine.Pin(15))

# Configuración del Relay (ventilador simulado)
relay = machine.Pin(2, machine.Pin.OUT)

# Configuración MQTT
mqtt_server = "broker.hivemq.com"
client_id = ubinascii.hexlify(machine.unique_id())
topic_pub_temp = b"home/livingroom/temperature"
topic_sub_fan = b"home/livingroom/fan"

def sub_callback(topic, msg):
    if msg == b'on':
        relay.value(1)
    elif msg == b'off':
        relay.value(0)

client = MQTTClient(client_id, mqtt_server)
client.set_callback(sub_callback)
client.connect()
client.subscribe(topic_sub_fan)

def read_sensor():
    dht_sensor.measure()
    temp = dht_sensor.temperature()
    return temp

while True:
    client.check_msg()
    temperature = read_sensor()
    client.publish(topic_pub_temp, str(temperature))
    time.sleep(10)  # Publica cada 10 segundos
```

Este script lee la temperatura del DHT22 y publica los datos en el tópico MQTT. Además, se suscribe a un tópico para encender/apagar el ventilador.

#### **2. Configuración de Node-RED**

Vamos a configurar Node-RED para recibir la temperatura, calcular el promedio cada minuto, y controlar el ventilador. También guardaremos los datos en una base de datos PostgreSQL.

#### **Paso 1: Flujo de Node-RED**

Copia y pega el siguiente flujo en tu Node-RED:

```json
[
    {
        "id": "ec5f2bdb46e6a6b7",
        "type": "tab",
        "label": "Control Ventilador",
        "disabled": false,
        "info": ""
    },
    {
        "id": "e1b1c4994c9a4bb4",
        "type": "mqtt in",
        "z": "ec5f2bdb46e6a6b7",
        "name": "Temperature Topic",
        "topic": "home/livingroom/temperature",
        "qos": "2",
        "datatype": "auto",
        "broker": "f5d5f0c5e43189f1",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 130,
        "y": 120,
        "wires": [
            [
                "b0e439684b882ad0"
            ]
        ]
    },
    {
        "id": "b0e439684b882ad0",
        "type": "function",
        "z": "ec5f2bdb46e6a6b7",
        "name": "Promedio 1 Minuto",
        "func": "var temperature = parseFloat(msg.payload);\nvar now = new Date().getTime();\n\ncontext.data = context.data || [];\ncontext.data.push({ temp: temperature, timestamp: now });\n\n// Filtrar datos más antiguos de 1 minuto\ncontext.data = context.data.filter(item => now - item.timestamp <= 60000);\n\nvar sum = context.data.reduce((acc, item) => acc + item.temp, 0);\nvar avg = sum / context.data.length;\n\nmsg.payload = avg;\n\n// Guardar la temperatura promedio en el contexto\nglobal.set('avgTemperature', avg);\n//Guardar la última temperatura registrada\nglobal.set('temp', temperature);\n\nreturn [msg];",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 360,
        "y": 20,
        "wires": [
            [
                "7471be6d95c4a4e0",
                "25b2c4557b8d61f3"
            ]
        ]
    },
    {
        "id": "7471be6d95c4a4e0",
        "type": "function",
        "z": "ec5f2bdb46e6a6b7",
        "name": "Guardar en PostgreSQL",
        "func": "var lastTemp = global.get('temp');\nmsg.payload = {\n    sensor: 1,             // Ejemplo: ID del sensor\n    temperature: lastTemp,     // Ejemplo: Valor de temperatura\n    user: 1                // Ejemplo: ID de usuario\n};\n\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 310,
        "y": 220,
        "wires": [
            [
                "5c937f74c8e7ab81"
            ]
        ]
    },
    {
        "id": "5c937f74c8e7ab81",
        "type": "postgresql",
        "z": "ec5f2bdb46e6a6b7",
        "name": "",
        "query": "INSERT INTO sensor_details (sensor_id, reading, user_id)\nVALUES ({{{msg.payload.sensor}}}, {{{msg.payload.temperature}}}, {{{msg.payload.user}}});\n",
        "postgreSQLConfig": "68157ea52b791d6b",
        "split": false,
        "rowsPerMsg": 1,
        "outputs": 1,
        "x": 590,
        "y": 260,
        "wires": [
            [
                "ddce8a0d06d0213b"
            ]
        ]
    },
    {
        "id": "ddce8a0d06d0213b",
        "type": "debug",
        "z": "ec5f2bdb46e6a6b7",
        "name": "",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 790,
        "y": 200,
        "wires": []
    },
    {
        "id": "934c29e813a8e65e",
        "type": "mqtt out",
        "z": "ec5f2bdb46e6a6b7",
        "name": "Fan State",
        "topic": "home/livingroom/fan",
        "qos": "",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "f5d5f0c5e43189f1",
        "x": 700,
        "y": 120,
        "wires": []
    },
    {
        "id": "25b2c4557b8d61f3",
        "type": "function",
        "z": "ec5f2bdb46e6a6b7",
        "name": "Controlar Ventilador",
        "func": "var avgTemperature = global.get('avgTemperature') || 0;\n\nif (avgTemperature > 25) {\n    return {