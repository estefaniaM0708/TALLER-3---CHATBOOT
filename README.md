# Chatbot con control por voz y ESP32

En esta actividad se realizó un **chatbot domótico orientado al encendido y apagado de un LED mediante comandos de voz**, tomando como referencia el chatbot presentado en el repositorio del curso.

La aplicación permite utilizar el micrófono del computador para reconocer las palabras **“encender”** y **“apagar”**. Dependiendo del comando identificado, el computador envía una instrucción hacia una ESP32 mediante comunicación serial y esta modifica el estado del LED conectado a uno de sus GPIO.

## ¿Cómo funciona el sistema?

Su funcionamiento de manera resumida se puede exponer en los siguientes pasos:

1. **Activación del micrófono:** El usuario presiona el botón `HABLAR` desde una página web.

2. **Reconocimiento de voz:** El navegador utiliza el micrófono para escuchar la instrucción del usuario.

3. **Identificación del comando:** El programa revisa si dentro de la frase reconocida se encuentra la palabra `encender` o `apagar`.

4. **Generación de la orden:** Dependiendo de la palabra detectada se genera un comando.

| Comando de voz | Orden |
| -------------- | ----- |
| Encender       | `ON`  |
| Apagar         | `OFF` |

5. **Comunicación serial:** Python envía el comando hacia la ESP32 mediante el puerto USB.

6. **Control del LED:** La ESP32 recibe la orden y cambia el estado del GPIO18.

El proceso general se puede representar como:

```text
Usuario
   ↓
Micrófono
   ↓
Reconocimiento de voz
   ↓
Aplicación Flask
   ↓
ON / OFF
   ↓
Comunicación serial
   ↓
ESP32
   ↓
GPIO18
   ↓
LED
```

# Desarrollo de la actividad

## 1. Revisión del chatbot

Inicialmente se revisó el chatbot disponible en el siguiente repositorio:

https://github.com/dialejobv/U_Militar/tree/main/3)%20chatbot

En este ejemplo se desarrolla un chatbot en Python que permite escribir mensajes desde la terminal y enviarlos hacia la API de **DeepSeek**.

Para ello se utiliza la librería:

```python
import requests
```

y se establece la dirección de la API:

```python
API_URL = 'https://api.deepseek.com/v1/chat/completions'
```

Posteriormente el programa recibe el mensaje del usuario:

```python
mensaje_usuario = input("Tú: ")
```

y lo envía hacia el servicio mediante una solicitud HTTP.

Para la actividad domótica se tomó como referencia esta idea de interacción entre el usuario y el programa. Sin embargo, se modificó el sistema para que la entrada no se realizara mediante texto, sino mediante **comandos de voz**, y la respuesta final fuera una acción física sobre la ESP32.

# 2. Aplicación del chatbot por voz

El programa `chatbot.py` se ejecuta en el computador y se encarga de crear la interfaz web, utilizar el reconocimiento de voz y establecer la comunicación con la ESP32.

Las librerías principales son:

```python
from flask import Flask, request, jsonify
import serial, time
```

Donde:

* `Flask` permite crear la aplicación web.
* `serial` permite comunicarse con la ESP32.
* `time` permite agregar tiempos de espera durante la ejecución.

La aplicación se inicializa mediante:

```python
app = Flask(__name__)
```

# 3. Comunicación con la ESP32

La comunicación entre el computador y el microcontrolador se realiza mediante:

```python
esp = serial.Serial("COM9", 115200, timeout=1)
time.sleep(2)
```

En esta implementación se utilizó:

```text
Puerto: COM9
Velocidad: 115200 baudios
```

El tiempo de espera permite que la ESP32 termine de iniciar después de abrir la conexión serial.

# 4. Interfaz del chatbot

La aplicación genera una página web sencilla que contiene un botón:

```html
<h2>Control ESP32 por voz</h2>
<button onclick="escuchar()">HABLAR</button>
<h3 id="texto">Esperando...</h3>
```

Al ejecutar el programa se crea un servidor local y desde el navegador se puede ingresar a:

```text
http://127.0.0.1:5000
```

Cuando se presiona el botón **HABLAR**, se activa el reconocimiento de voz del navegador.

# 5. Reconocimiento de voz

Para utilizar el micrófono se implementó:

```javascript
let R = window.SpeechRecognition || window.webkitSpeechRecognition;
let r = new R();
```

Posteriormente se configuró el idioma:

```javascript
r.lang = "es-CO";
```

Esto permite que el sistema interprete comandos pronunciados en español.

Cuando se activa el micrófono aparece:

```text
Escuchando...
```

Después de reconocer la voz, la frase obtenida se guarda mediante:

```javascript
let voz = e.results[e.results.length-1][0].transcript.toLowerCase();
```

El texto se convierte a minúsculas para facilitar la búsqueda de las palabras clave.

Por ejemplo, si el usuario dice:

```text
Encender el LED
```

el programa obtiene:

```text
encender el led
```

# 6. Identificación de los comandos

Después de convertir la voz en texto se verifica si aparece la palabra `encender`.

```javascript
if(voz.includes("encender")){
    enviar("ON");
    r.stop();
}
```

Cuando se detecta esta palabra se genera el comando:

```text
ON
```

Para apagar se utiliza:

```javascript
if(voz.includes("apagar")){
    enviar("OFF");
    r.stop();
}
```

generando:

```text
OFF
```

El programa no requiere que el usuario diga exactamente una única palabra. También puede reconocer frases como:

```text
Encender el LED
Quiero encender la luz
Apagar el LED
Por favor apagar la luz
```

siempre que contengan las palabras configuradas.

# 7. Envío del comando hacia Flask

Después de reconocer la orden se utiliza `fetch()` para enviar el comando desde la página web hacia Python.

```javascript
fetch("/comando", {
    method:"POST",
    headers:{"Content-Type":"application/json"},
    body:JSON.stringify({cmd:cmd})
})
```

Flask recibe la información mediante:

```python
@app.route("/comando", methods=["POST"])
def comando():
    cmd = request.json["cmd"]
```

Posteriormente el comando es enviado hacia la ESP32:

```python
esp.write((cmd + "\n").encode())
```

De esta manera, la instrucción reconocida mediante el micrófono termina convertida en información serial que puede interpretar el microcontrolador.

# 8. Programa de la ESP32

El segundo programa corresponde a `main.py` y se ejecuta directamente en la ESP32 mediante **MicroPython**.

Primero se importan los elementos necesarios:

```python
from machine import Pin
import sys
```

Posteriormente se configura el LED como salida:

```python
led = Pin(18, Pin.OUT)
led.off()
```

Por lo tanto, el circuito utiliza:

| Elemento       | Conexión       |
| -------------- | -------------- |
| LED            | GPIO18         |
| Tipo de GPIO   | Salida digital |
| Estado inicial | Apagado        |

Al iniciar el programa la ESP32 muestra:

```python
print("ESP_LISTA")
```

indicando que está preparada para recibir comandos.

# 9. Recepción de los comandos

La ESP32 permanece constantemente esperando información proveniente del computador:

```python
while True:
    cmd = sys.stdin.readline().strip().upper()
```

La función `readline()` recibe la información serial y `upper()` convierte el comando a mayúsculas.

De esta manera, el programa puede comparar fácilmente si recibió:

```text
ON
OFF
STATUS
```

# 10. Encendido del LED

Cuando la ESP32 recibe:

```text
ON
```

se ejecuta:

```python
if cmd == "ON":
    led.on()
    print("OK: LED ENCENDIDO")
```

El GPIO18 pasa a nivel alto y el LED se enciende.

La ESP32 también devuelve el mensaje:

```text
OK: LED ENCENDIDO
```

para confirmar que la instrucción fue realizada.

# 11. Apagado del LED

Cuando se recibe:

```text
OFF
```

se ejecuta:

```python
elif cmd == "OFF":
    led.off()
    print("OK: LED APAGADO")
```

El GPIO18 pasa a nivel bajo y el LED se apaga.

La respuesta enviada hacia el computador es:

```text
OK: LED APAGADO
```

# 12. Consulta del estado

El programa también permite utilizar:

```text
STATUS
```

para consultar el estado actual del LED.

```python
elif cmd == "STATUS":
    print("ESTADO: " + ("ENCENDIDO" if led.value() else "APAGADO"))
```

Dependiendo del estado del GPIO se obtiene:

```text
ESTADO: ENCENDIDO
```

o:

```text
ESTADO: APAGADO
```

Si la ESP32 recibe un comando diferente a los anteriores, devuelve:

```python
else:
    print("ERROR: COMANDO DESCONOCIDO")
```

# 13. Respuesta hacia la página web

Después de enviar el comando, Python espera la respuesta generada por la ESP32:

```python
respuesta = esp.readline().decode().strip()
```

Luego Flask la devuelve hacia la interfaz:

```python
return jsonify(respuesta=respuesta)
```

Por esta razón, después de decir **encender**, la página puede mostrar:

```text
OK: LED ENCENDIDO
```

y después de decir **apagar**:

```text
OK: LED APAGADO
```

Esto permite comprobar que la ESP32 recibió y ejecutó correctamente la orden.

# 14. Integración completa

El funcionamiento completo para encender el LED puede representarse de la siguiente manera:

```text
Usuario dice "encender"
          ↓
Micrófono
          ↓
SpeechRecognition
          ↓
Detecta la palabra "encender"
          ↓
Comando ON
          ↓
Flask
          ↓
Comunicación serial COM9
          ↓
ESP32
          ↓
GPIO18 = HIGH
          ↓
LED ENCENDIDO
```

Para apagar:

```text
Usuario dice "apagar"
          ↓
Micrófono
          ↓
SpeechRecognition
          ↓
Detecta la palabra "apagar"
          ↓
Comando OFF
          ↓
Flask
          ↓
Comunicación serial COM9
          ↓
ESP32
          ↓
GPIO18 = LOW
          ↓
LED APAGADO
```

De esta manera, el sistema integra reconocimiento de voz, una aplicación web, comunicación serial y un sistema embebido.

# 15. Procedimiento realizado

Para desarrollar la actividad se realizaron los siguientes pasos:

1. Se revisó el chatbot presentado en el repositorio de referencia.

2. Se configuró la ESP32 para trabajar con MicroPython.

3. Se conectó el LED al GPIO18 de la ESP32 utilizando su correspondiente resistencia limitadora de corriente.

4. Se creó el archivo `main.py` encargado de recibir los comandos y controlar el LED.

5. Se cargó `main.py` dentro de la ESP32.

6. Se creó el archivo `chatbot.py` en Visual Studio Code.

7. Se instalaron las librerías necesarias:

```bash
pip install flask pyserial
```

8. Se conectó la ESP32 al computador mediante USB.

9. Se identificó el puerto correspondiente, que para esta práctica fue:

```text
COM9
```

10. Se ejecutó la aplicación mediante:

```bash
python chatbot.py
```

11. Desde el navegador se abrió:

```text
http://127.0.0.1:5000
```

12. Se permitió el acceso al micrófono.

13. Se presionó el botón **HABLAR**.

14. Al pronunciar **“encender”**, el programa generó `ON` y encendió el LED.

15. Al pronunciar **“apagar”**, se generó `OFF` y el LED se apagó.

# Resultado

Como resultado se obtuvo un sistema domótico básico capaz de controlar físicamente un LED mediante **comandos de voz**.

El navegador reconoce la instrucción pronunciada por el usuario y la aplicación desarrollada en Python transforma esta información en los comandos `ON` y `OFF`.

Posteriormente estos comandos son enviados hacia la ESP32 mediante comunicación serial. Finalmente, el microcontrolador modifica el estado del GPIO18 y devuelve una confirmación hacia la aplicación.

El proceso permitió integrar:

```text
Reconocimiento de voz
        +
Aplicación web
        +
Python
        +
Flask
        +
Comunicación serial
        +
MicroPython
        +
ESP32
```

# Video de evidencia

A continuación se presenta el video donde se puede observar el funcionamiento final del chatbot domótico.

En la evidencia se muestra cómo el usuario utiliza el micrófono para dar las instrucciones **“encender”** y **“apagar”**, mientras la ESP32 recibe los comandos y modifica físicamente el estado del LED.

### [Ver video de evidencia en Google Drive](https://drive.google.com/file/d/17j4eG9SLA5o9AJOmKrW2_eyuFpYh_WF3/view?usp=sharing)

# Conclusión

Con el desarrollo de esta actividad se logró implementar un sistema básico de **domótica controlado mediante comandos de voz**. El reconocimiento realizado desde el navegador permitió convertir las instrucciones pronunciadas por el usuario en órdenes digitales que posteriormente fueron enviadas hacia la ESP32.

La aplicación Flask funcionó como intermediario entre la interfaz web y el microcontrolador, mientras que MicroPython permitió controlar directamente el LED conectado al GPIO18.

Aunque para esta práctica se utilizó únicamente un LED, el mismo principio puede ampliarse para controlar dispositivos como lámparas, relés, ventiladores, motores o diferentes cargas utilizadas dentro de un sistema domótico.

# Recursos

* Repositorio de referencia:
  https://github.com/dialejobv/U_Militar/tree/main/3)%20chatbot

* Video de evidencia:
  https://drive.google.com/file/d/17j4eG9SLA5o9AJOmKrW2_eyuFpYh_WF3/view?usp=sharing

* MicroPython.

* ESP32.

* Visual Studio Code.

* Reconocimiento de voz del navegador.

