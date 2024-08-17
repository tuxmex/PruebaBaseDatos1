Aquí está la versión modificada del flujo de Node-RED para calcular el promedio de temperatura en un intervalo de 1 minuto en lugar de 30 minutos:

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
        "func": "var temperature = parseFloat(msg.payload);\nvar now = new Date().getTime();\n\ncontext.data = context.data || [];\ncontext.data.push({ temp: temperature, timestamp: now });\n\n// Filtrar datos más antiguos de 1 minuto\ncontext.data = context.data.filter(item => now - item.timestamp <= 60000);\n\nvar sum = context.data.reduce((acc, item) => acc + item.temp, 0);\nvar avg = sum / context.data.length;\n\nmsg.payload = avg;\n\n// Guardar la temperatura promedio en el contexto\nglobal.set('avgTemperature', avg);\n//Guardar la ultima temperatura registrada\nglobal.set('temp', temperature);\n\nreturn [msg];",
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
                "ddce8a0d06d0213b",
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
                "5c937f74c8e7ab81",
                "ddce8a0d06d0213b"
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
        "id": "f5d5f0c5e43189f1",
        "type": "mqtt-broker",
        "name": "HiveMQ",
        "broker": "broker.hivemq.com",
        "port": "1883",
        "clientid": "",
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthPayload": "",
        "closeTopic": "",
        "closeQos": "0",
        "closePayload": "",
        "willTopic": "",
        "willQos": "0",
        "willPayload": ""
    },
    {
        "id": "68157ea52b791d6b",
        "type": "postgresql-config",
        "name": "PostgreSQL",
        "host": "your_postgresql_host",
        "host_field": "localhost",
        "ssl": false,
        "port": "5432",
        "database": "your_database_name",
        "name_field": "your_database_name",
        "username": "your_username",
        "password": "your_password",
        "encoding": "utf-8"
    }
]
```

### **Modificaciones realizadas:**
1. **Intervalo de Promedio**: El cálculo del promedio de temperatura se ha reducido a un intervalo de 1 minuto (60000 ms).
2. **Conexión MQTT**: Conecta el nodo `mqtt out` al estado del ventilador y el `mqtt in` a la temperatura para manejar la lógica.

Este flujo puede ser importado directamente en Node-RED para probar el comportamiento del sistema en Wokwi. Asegúrate de que la configuración de tu base de datos PostgreSQL esté correctamente definida en el nodo `postgresql-config`.