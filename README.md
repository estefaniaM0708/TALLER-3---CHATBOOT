# Chatbot con control por voz y ESP32

En esta actividad se realizó un **chatbot domótico orientado al control de un LED mediante comandos de voz**, tomando como referencia la estructura del chatbot presentado en el repositorio del curso.

El sistema permite que el usuario utilice el micrófono del computador para decir las palabras **“encender”** o **“apagar”**. El comando reconocido es enviado desde una aplicación desarrollada en Python hacia una **ESP32**, que finalmente controla el estado físico del LED.

# ¿Cómo funciona el sistema?

Su funcionamiento de manera resumida se puede exponer en los siguientes pasos:

1. **Ingreso del usuario:** El usuario presiona el botón **HABLAR** desde una página web.

2. **Reconocimiento de voz:** El navegador utiliza el micrófono para escuchar lo que dice el usuario.

3. **Identificación del comando:** El programa verifica si dentro de la frase aparece la palabra `encender` o `apagar`.

4. **Generación de la orden:** Dependiendo de la palabra detectada, se genera uno de los siguientes comandos:

| Comando de voz | Orden enviada |
| -------------- | ------------- |
| Encender       | `ON`          |
| Apagar         | `OFF`         |

5. **Comunicación serial:** Python envía la orden mediante USB hacia la ESP32.

6. **Control de la salida:** La ESP32 recibe la información y modifica el estado del GPIO18.

El funcionamiento general se puede representar como:

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

## 1. Chatbot de referencia

Inicialmente se revisó el chatbot presentado en el repositorio:

https://github.com/dialejobv/U_Militar/tree/main/3)%20chatbot

El ejemplo utiliza Python para recibir un mensaje escrito por el usuario y enviarlo hacia la API de **DeepSeek**.

En el código original se utiliza:

```python
import requests
```

para realizar la solicitud HTTP hacia:

```python
API_URL = 'https://api.deepseek.com/v1/chat/completions'
```

Posteriormente se envía el mensaje escrito por el usuario y se presenta la respuesta obtenida desde el modelo.

Para la actividad domótica se tomó como referencia esta estructura de interacción, pero se cambió la entrada de texto por una **entrada mediante voz** y la respuesta del sistema se orientó al control físico de un LED mediante la ESP32.

# 2. Aplicación de control por voz

El archivo `chatbot.py` se ejecuta en el computador y contiene la aplicación encargada de recibir los comandos de voz.

Para su funcionamiento se utilizan principalmente:

```python
from flask import Flask, request, jsonify
import serial, time
```

Donde:

* `Flask` permite crear la aplicación web.
* `serial` permite establecer la comunicación con la ESP32.
* `time` se utiliza para controlar los tiempos de inicialización y respuesta.

## Comunicación con la ESP32

La conexión se realiza mediante:

```python
esp = serial.Serial("COM9", 115200, timeout=1)
```

Por lo tanto, para esta implementación se utilizó:

```text
Puerto: COM9
Velocidad: 115200 baudios
```

Después de abrir el puerto se espera aproximadamente dos segundos para permitir que la ESP32 termine su proceso de inicialización.

# 3. Interfaz del chatbot

La aplicación Flask genera una página web sencilla con un botón:

```html
<button onclick="escuchar()">HABLAR</button>
```

Cuando el usuario lo presiona, el sistema activa el reconocimiento de voz.

En pantalla aparece inicialmente:

```text
Escuchando...
```

y posteriormente se muestra la frase reconocida por el navegador.

Por ejemplo:

```text
Escuché: encender la luz
```

# 4. Reconocimiento de voz

El reconocimiento se realiza mediante:

```javascript
let R = window.SpeechRecognition || window.webkitSpeechRecognition;
let r = new R();
```

Posteriormente se establece el idioma:

```javascript
r.lang = "es-CO";
```

De esta manera, el sistema queda preparado para reconocer comandos pronunciados en español de Colombia.

El programa escucha la frase y la convierte a minúsculas:

```javascript
let voz = e.results[e.results.length-1][0].transcript.toLowerCase();
```

Luego comprueba las palabras importantes.

Para encender:

```javascript
if(voz.includes("encender")){
    enviar("ON");
    r.stop();
}
```

Para apagar:

```javascript
if(voz.includes("apagar")){
    enviar("OFF");
    r.stop();
}
```

Por esta razón no es necesario decir únicamente una palabra exacta. El sistema también puede reconocer expresiones como:

```text
Encender el LED
Quiero encender la luz
Apagar el LED
Por favor apagar la luz
```

Siempre que la frase contenga la palabra esperada.

# 5. Envío del comando hacia Python

Cuando se reconoce una orden, JavaScript utiliza `fetch()` para enviar la información hacia la aplicación Flask:

```javascript
fetch("/comando", {
    method:"POST",
    headers:{"Content-Type":"application/json"},
    body:JSON.stringify({cmd:cmd})
})
```

Por ejemplo, si se reconoce la palabra **encender**, se envía:

```text
ON
```

Si se reconoce **apagar**, se envía:

```text
OFF
```

Flask recibe esta información mediante:

```python
@app.route("/comando", methods=["POST"])
def comando():
    cmd = request.json["cmd"]
```

Posteriormente el comando se envía hacia la ESP32:

```python
esp.write((cmd + "\n").encode())
```

# 6. Programa de la ESP32

El archivo `main.py` se ejecuta directamente en la ESP32 mediante **MicroPython**.

Inicialmente se configura el LED:

```python
from machine import Pin
import sys

led = Pin(18, Pin.OUT)
led.off()
```

En esta implementación el LED está conectado al:

```text
GPIO18
```

Al iniciar el programa permanece apagado.

La ESP32 entra posteriormente en un ciclo continuo esperando información proveniente del computador:

```python
while True:
    cmd = sys.stdin.readline().strip().upper()
```

El comando recibido se convierte a mayúsculas para evitar problemas en su interpretación.

# 7. Interpretación de los comandos

Cuando se recibe:

```text
ON
```

se ejecuta:

```python
if cmd == "ON":
    led.on()
    print("OK: LED ENCENDIDO")
```

La salida GPIO18 pasa a nivel alto y el LED se enciende.

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

El GPIO18 cambia a nivel bajo y el LED se apaga.

También se implementó el comando:

```text
STATUS
```

que permite consultar el estado del LED:

```python
elif cmd == "STATUS":
    print("ESTADO: " + ("ENCENDIDO" if led.value() else "APAGADO"))
```

En caso de recibir una orden que no corresponda con las anteriores, la ESP32 informa:

```text
ERROR: COMANDO DESCONOCIDO
```

# 8. Respuesta de la ESP32

Una vez ejecutado el comando, la ESP32 devuelve una respuesta mediante comunicación serial.

Python espera esta información:

```python
respuesta = esp.readline().decode().strip()
```

Posteriormente la respuesta se envía nuevamente hacia la página web:

```python
return jsonify(respuesta=respuesta)
```

De esta manera, cuando el LED se enciende, el usuario puede observar:

```text
OK: LED ENCENDIDO
```

y cuando se apaga:

```text
OK: LED APAGADO
```

Esto permite tener una confirmación de que la orden realmente fue recibida por la ESP32.

# 9. Integración completa

La comunicación entre todos los elementos puede representarse de la siguiente manera:

```text
Usuario dice "encender"
           ↓
Micrófono del computador
           ↓
SpeechRecognition
           ↓
Detecta "encender"
           ↓
Genera comando ON
           ↓
Flask
           ↓
Puerto serial COM9
           ↓
ESP32
           ↓
GPIO18 = HIGH
           ↓
LED ENCENDIDO
```

Para apagar ocurre el proceso contrario:

```text
Usuario dice "apagar"
           ↓
SpeechRecognition
           ↓
Genera comando OFF
           ↓
Flask
           ↓
Puerto serial COM9
           ↓
ESP32
           ↓
GPIO18 = LOW
           ↓
LED APAGADO
```

# 10. Procedimiento realizado

Para desarrollar la actividad se realizaron los siguientes pasos:

1. Se revisó el código del chatbot de referencia disponible en el repositorio.

2. Se configuró la ESP32 con MicroPython.

3. Se conectó un LED al **GPIO18** de la ESP32 utilizando su respectiva resistencia.

4. Se cargó `main.py` dentro de la ESP32.

5. En Visual Studio Code se creó el programa `chatbot.py`.

6. Se instalaron las librerías necesarias:

```bash
pip install flask pyserial
```

7. Se identificó el puerto utilizado por la ESP32, correspondiente a **COM9**.

8. Se ejecutó el servidor mediante:

```bash
python chatbot.py
```

9. La aplicación quedó disponible localmente en:

```text
http://127.0.0.1:5000
```

10. Desde el navegador se presionó el botón **HABLAR**.

11. Se permitió el acceso al micrófono.

12. Al decir **“encender”**, el sistema envió `ON` y activó el LED.

13. Al decir **“apagar”**, se envió `OFF` y el LED se desactivó.

# Resultado

Como resultado se obtuvo un sistema domótico capaz de controlar una salida física de la ESP32 utilizando únicamente comandos de voz.

La aplicación reconoce las palabras **encender** y **apagar**, las transforma en comandos digitales y posteriormente las envía hacia la ESP32 mediante comunicación serial.

El sistema permitió comprobar la integración entre:

```text
Reconocimiento de voz
+
Aplicación web
+
Python
+
Comunicación serial
+
MicroPython
+
ESP32
```

# Conclusión

Con el desarrollo de esta actividad se implementó un sistema básico de **domótica controlado mediante voz**. El navegador se encargó de reconocer las instrucciones pronunciadas por el usuario, mientras que Flask permitió procesarlas y comunicarlas con la ESP32.

La ESP32 recibió los comandos mediante el puerto serial y realizó el control físico del LED conectado al GPIO18.

Aunque la práctica utiliza únicamente un LED, el mismo principio puede aplicarse posteriormente para controlar iluminación, relés, ventiladores, motores u otros dispositivos utilizados en sistemas domóticos.

## Recursos utilizados

* Repositorio de referencia:
  https://github.com/dialejobv/U_Militar/tree/main/3)%20chatbot

* Python.

* Flask.

* PySerial.

* MicroPython.

* ESP32.

* Reconocimiento de voz del navegador.
