# Proyecto LM35 - Monitoreo de Temperatura con ThingSpeak y MicroPython

Este proyecto permite medir la temperatura utilizando un sensor **LM35** conectado a una **Raspberry Pi Pico W**. Los datos de temperatura se envían a ThingSpeak, donde se pueden visualizar y analizar. Además, el proyecto calcula los promedios de cada bloque de 10 datos consecutivos para proporcionar una vista más estable y continua de las lecturas de temperatura.

## Descripción

El sensor LM35 es un sensor de temperatura analógico que genera una salida de voltaje proporcional a la temperatura. Cada grado Celsius se traduce en un aumento de **10 mV**. Con una Raspberry Pi Pico W, el sensor se conecta al puerto ADC0 (GPIO26) y los datos se leen a través del convertidor analógico a digital (ADC). Estos datos son luego enviados a ThingSpeak para su visualización en tiempo real.

Funcionalidades:
 Lectura continua de temperatura desde el LM35.
 Envío de los datos de temperatura a ThingSpeak.
 Cálculo del promedio de cada bloque de 10 datos consecutivos.
 Visualización de los promedios en gráficos interactivos usando MathWorks en ThingSpeak.
 Envío de una alerta por correo electrónico si la temperatura promedio supera los 35°C.

   Requisitos

  Hardware:
  - Raspberry Pi Pico W
  - Sensor de temperatura LM35
  - Cables para la conexión

- **Software:**
  - MicroPython en la Raspberry Pi Pico W
  - Conexión WiFi
  - Cuenta en **ThingSpeak**
  - Editor de código compatible con MicroPython (como Thonny)
  
- **Bibliotecas necesarias:**
  - `urequests` para enviar datos a ThingSpeak.
  - `usmtplib` para enviar correos electrónicos (si es necesario configurar alertas).

## Instalación

### Paso 1: Configurar la Raspberry Pi Pico W

1. Descarga e instala **MicroPython** en la Raspberry Pi Pico W.
2. Conecta el sensor **LM35** al pin **ADC0** (GPIO26) de la Raspberry Pi Pico W.
3. Conecta la Raspberry Pi Pico W a tu computadora.

### Paso 2: Preparar el código

1. Abre el editor de **Thonny** (o cualquier otro IDE compatible con MicroPython).
2. Copia el siguiente código al editor de Thonny y cárgalo a la Raspberry Pi Pico W:

```python
import time
import network
import urequests
import usmtplib  # Usamos usmtplib para enviar correos
from machine import Pin, ADC

# Configuración del ADC (LM35 está en ADC0)
adc = ADC(Pin(26))  # ADC0 está en el pin GPIO26 en la Raspberry Pi Pico W

# Función para conectar al WiFi
def connect_wifi(ssid, password):
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('Conectando al WiFi...')
        wlan.connect(ssid, password)
        while not wlan.isconnected():
            time.sleep(1)
    print('Conexión establecida. Dirección IP:', wlan.ifconfig()[0])

# Función para leer la temperatura del LM35 (conversión de ADC a temperatura)
def read_temperature():
    voltage = adc.read_u16() * (3.3 / 65535)  # Lectura de 16 bits (0-65535)
    temperature_celsius = voltage * 100  # Convertir el voltaje a temperatura (10mV/°C)
    return temperature_celsius

# Función para enviar el correo electrónico de alerta
def send_alert_email(subject, body):
    import usmtplib
    from email.mime.text import MIMEText
    from email.mime.multipart import MIMEMultipart

    sender_email = "tu_correo@gmail.com"  # Reemplaza con tu correo
    receiver_email = "correo_destino@gmail.com"  # Reemplaza con el correo destino
    password = "tu_contraseña"  # Contraseña o contraseña de aplicación de Gmail

    # Configura el servidor SMTP de Gmail
    server = usmtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()  # Inicia la conexión segura
    server.login(sender_email, password)

    # Crea el mensaje
    message = MIMEMultipart()
    message['From'] = sender_email
    message['To'] = receiver_email
    message['Subject'] = subject
    message.attach(MIMEText(body, 'plain'))

    # Enviar el correo
    server.sendmail(sender_email, receiver_email, message.as_string())
    print("Correo de alerta enviado.")
    server.quit()

# Función para enviar los datos a ThingSpeak
def send_to_thingspeak(temperature):
    api_key = 'TU_API_KEY'  # Reemplaza con tu API key de ThingSpeak
    url = f'https://api.thingspeak.com/update?api_key={api_key}&field1={temperature}'
    
    try:
        response = urequests.get(url)
        if response.status_code == 200:
            print("Datos enviados correctamente a ThingSpeak.")
        else:
            print("Error al enviar los datos.")
        response.close()
    except Exception as e:
        print("Error de conexión:", e)

# Conectar a WiFi
ssid = 'TU_SSID'
password = 'TU_PASSWORD'
connect_wifi(ssid, password)

# Bucle principal
while True:
    # Leer la temperatura del LM35
    temperature = read_temperature()
    print('Temperatura: {:.2f} °C'.format(temperature))
    
    # Enviar la temperatura a ThingSpeak
    send_to_thingspeak(temperature)
    
    # Si la temperatura supera los 35°C, enviar un correo de alerta
    if temperature > 35:
        send_alert_email(
            subject="¡Alerta de Temperatura Alta!",
            body=f"La temperatura ha superado los 35°C. Temperatura actual: {temperature:.2f}°C"
        )
    
    # Esperar 60 segundos antes de enviar el siguiente dato
    time.sleep(60)
