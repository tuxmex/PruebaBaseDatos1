Claro, aquí está el tutorial completo con los cambios realizados.

---

### Tutorial Completo: Control de Ventilador con ESP32, DHT22, Node-RED y PostgreSQL

Este tutorial te guiará a través de la creación de un sistema para controlar un ventilador usando un ESP32 con un sensor DHT22, procesando los datos en Node-RED, y almacenando las lecturas de temperatura en una base de datos PostgreSQL. El sistema también simulará el ventilador con un LED y un relé.

#### **1. Configuración en Wokwi**

**1.1. Configuración del Circuito:**

1. **DHT22 Sensor:**
   - **VCC** a **3.3V** del ESP32.
   - **GND** a **GND** del ESP32.
   - **DATA** a **GPIO15** en el ESP32 (puedes usar cualquier pin GPIO disponible).

2. **Relay y LED:**
   - **VCC** del relay a **3.3V** del ESP32.
   - **GND** del relay a **GND** del ESP32.
   - **Pin de señal** del relay a **GPIO2** del ESP32.
   - Conecta el LED en serie con el relay. Una pierna del LED al pin **NO (Normally Open)** del relay y la otra al **GND** del ESP32, usando una resistencia adecuada.

**1.2. Código MicroPython para el ESP32:**

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

#### **2. Configuración de Node-RED**

**2.1. Flujo de Node-RED:**

Aquí tienes el flujo de Node-RED actualizado para manejar la temperatura, calcular el promedio cada minuto, y controlar el ventilador.

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
        "func": "var avgTemperature = global.get('avgTemperature') || 0;\n\nif (avgTemperature > 25) {\n    return { payload: \"on\" };\n} else {\n    return { payload: \"off\" };\n}",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 140,
        "wires": [
            [
                "934c29e813a8e65e",
                "ddce8a0d06d0213b"
            ]
        ]
    },
    {
        "id": "f5d5f0c5e43189f1",
        "type": "mqtt-broker",
        "name": "Mosquitto Broker",
        "broker": "broker.hivemq.com",
        "port": "1883",


        "tls": "",
        "clientid": "",
        "usetls": false,
        "verifyservercert": true,
        "maxqueue": "1000",
        "retain": false,
        "clean": true,
        "username": "",
        "password": "",
        "protocol": "mqtt",
        "keepalive": "60",
        "sessionExpiryInterval": "",
        "willTopic": "",
        "willQos": "",
        "willPayload": "",
        "willRetain": ""
    },
    {
        "id": "68157ea52b791d6b",
        "type": "postgresql",
        "name": "PostgreSQL Database",
        "host": "localhost",
        "port": "5432",
        "database": "your_database",
        "user": "your_user",
        "password": "your_password"
    }
]
```

**2.2. Descripción del Flujo:**

1. **MQTT In Node:**
   - **Topic:** `home/livingroom/temperature`
   - Lee los datos de temperatura del tópico MQTT.

2. **Function Node - Promedio 1 Minuto:**
   - Calcula el promedio de temperatura en los últimos 60 segundos.

3. **Function Node - Guardar en PostgreSQL:**
   - Prepara los datos para ser insertados en la base de datos PostgreSQL.

4. **PostgreSQL Node:**
   - Inserta los datos de temperatura en la base de datos.

5. **Function Node - Controlar Ventilador:**
   - Decide si el ventilador debe estar encendido o apagado basado en el promedio de temperatura.

6. **MQTT Out Node:**
   - Publica el estado del ventilador en el tópico MQTT para que el ESP32 pueda actuar en consecuencia.

7. **Debug Node:**
   - Muestra los datos de salida en la consola de depuración de Node-RED.

#### **3. Configuración de la Base de Datos PostgreSQL**

**3.1. Creación de la Tabla:**

Ejecuta el siguiente comando SQL para crear la tabla en tu base de datos PostgreSQL:

```sql
CREATE TABLE sensor_details (
    id SERIAL PRIMARY KEY,
    sensor_id INTEGER,
    reading FLOAT,
    user_id INTEGER,
    timestamp TIMESTAMPTZ DEFAULT NOW()
);
```

Asegúrate de reemplazar `your_database`, `your_user`, y `your_password` en la configuración del nodo PostgreSQL con tus credenciales reales.

#### **4. Integración y Prueba**

1. **Configura el broker MQTT** en Node-RED con la misma configuración que usas en el código MicroPython.
2. **Ejecuta el código MicroPython** en tu ESP32.
3. **Importa el flujo de Node-RED** y asegúrate de que todo esté correctamente conectado.
4. **Verifica la base de datos PostgreSQL** para asegurarte de que los datos se estén almacenando correctamente.
5. **Prueba el control del ventilador** encendiendo y apagando el LED desde Node-RED y observando los resultados en la simulación de Wokwi.

¡Y eso es todo! Ahora tienes un sistema completo para monitorear y controlar un ventilador basado en la temperatura utilizando un ESP32, un DHT22, Node-RED y PostgreSQL.