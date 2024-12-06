# ProyectoNavideno

# Personaje2024

## Nombre del personaje
Nacimiento del niño dios con musice e iluminación.
## Creador
Luis Manuel López Cano
## Explicacion del funcionamiento
El proyecto contara con un sensor, una placa escp32, un buzzer, motor a paso, leds y resistencias. El proyecto consiste en que en el momento que se hacer una persona en un rango de un metro, el sensor va activar la cancion que reproducira el buzzer la cual es "CAMPANAS DE BELEN" y con esta se encienden las diez ledsw en grupos de 5 y en secuencia como si fuera una serie de luces navideña y el motor paso a paso hara girar a un angel.

## Materiales a utlizar
|Material|Imagen|Cantidad|Costo|
|--|--|--|--|
|ESP32|<img src="https://m.media-amazon.com/images/I/612eALAbpgL.jpg" width="100"/>|1|120.00|
|HC-SR04|<img width="100" src="https://www.330ohms.com/cdn/shop/products/photo_A_OS-03261_SensorUltrasonico_HC-SR04_01_1200x1200.png?v=1598042103" />|1|30.00|
|Buzzer| <img width="100" src="https://m.media-amazon.com/images/I/51ywawraqaL._SX466_.jpg" />|1|30.00|";
|LEDs|<img width="100" src="https://www.taloselectronics.com/cdn/shop/products/paquete_de_100_leds_difusos_5mm_varios_colores_mexico_jalisco_guadalajara_700x700.jpg?v=1593816653" />|10|1.00|
|Resistencia|<img width="100" src="https://http2.mlstatic.com/D_NQ_NP_903666-MLM75952546015_042024-O.webp" />|10|1.00|
|Motor a pasos|<img width="100" src="https://uelectronics.com/wp-content/uploads/2017/08/AR0130-Motor-a-pasos-28BYJ-48-V1.jpg" />|1|60.00|
|...||||

## Software a utilizar
|Software|Versión|
|--|--|
|Thonny|4.1.6|
|...||

## Código de Thonny
|--|
import _thread
from hcsr04 import HCSR04
from machine import Pin, PWM
from time import sleep, sleep_us
import network
from umqtt.simple import MQTTClient

# Propiedades para conectar a un cliente MQTT
MQTT_BROKER = "broker.emqx.io"
MQTT_USER = ""
MQTT_PASSWORD = ""
MQTT_CLIENT_ID = ""
MQTT_TOPIC = "manuelitogg"
MQTT_PORT = 1883

# Estado para controlar activación por MQTT
estado_mqtt = False
melody_thread_active = False

# Función para conectar a WiFi
def conectar_wifi():
    print("Conectando...", end="")
    sta_if = network.WLAN(network.STA_IF)
    sta_if.active(True)
    sta_if.connect('UTNG_GUEST', 'R3d1nv1t4d0s#UT')
    while not sta_if.isconnected():
        print(".", end="")
        sleep(0.3)
    print("\nWiFi Conectada!")

# Configuración de los LEDs
led_pins = [15, 2, 4, 16, 17, 5, 18, 19, 21, 23]
leds = [Pin(pin, Pin.OUT) for pin in led_pins]

# Configuración del buzzer
buzzer = PWM(Pin(26))
buzzer.duty_u16(0)  # Inicia el buzzer apagado

# Notas musicales
notes = {
    "B3": 123,
    "C4": 131,
    "D4": 147,
    "E4": 165,
    "F4": 174,
    "G4": 196,
    "A4": 220,
    "B4": 247,
    "C5": 262,
    "D5": 294,
    "E5": 330,
    "F5": 349,
    "G5": 392,
    "A5": 440,
    "B5": 494,
    "C6": 523,
    "REST": 0
}

# Melodía de "Campanas de Belén"
campanas_belen = [
    ("E4", 0.5), ("F4", 0.5), ("G4", 1),
    ("E4", 0.5), ("F4", 0.5), ("G4", 1),
    ("G4", 0.25), ("A4", 0.25), ("G4", 0.25), ("F4", 0.25), ("E4", 1),
    ("F4", 0.5), ("G4", 0.5), ("E4", 1),

    ("E4", 0.5), ("F4", 0.5), ("G4", 1),
    ("E4", 0.5), ("F4", 0.5), ("G4", 1),
    ("G4", 0.25), ("A4", 0.25), ("G4", 0.25), ("F4", 0.25), ("E4", 1),
    ("F4", 0.5), ("G4", 0.5), ("E4", 1),
]

# Función para tocar una nota
def play_tone(note, duration):
    if note == "REST":
        buzzer.duty_u16(0)  # Silencio
    else:
        buzzer.freq(notes[note])
        buzzer.duty_u16(45000)  # Volumen alto
    sleep(duration)
    buzzer.duty_u16(0)  # Apagar al finalizar

# Función para tocar la melodía
def play_melody():
    for note, duration in campanas_belen:
        play_tone(note, duration)

# Configuración del motor paso a paso
IN1 = Pin(13, Pin.OUT)
IN2 = Pin(12, Pin.OUT)
IN3 = Pin(14, Pin.OUT)
IN4 = Pin(27, Pin.OUT)

# Secuencia del motor paso a paso
secuencia_horario = [
    [1, 0, 0, 1],
    [1, 0, 0, 0],
    [1, 1, 0, 0],
    [0, 1, 0, 0],
    [0, 1, 1, 0],
    [0, 0, 1, 0],
    [0, 0, 1, 1],
    [0, 0, 0, 1]
]

# Función para mover el motor paso a paso
def mover_motor(secuencia, pasos, delay_us):
    for _ in range(pasos):
        for paso in secuencia:
            IN1.value(paso[0])
            IN2.value(paso[1])
            IN3.value(paso[2])
            IN4.value(paso[3])
            sleep_us(delay_us)

# Función para apagar todo
def apagar_todo():
    global melody_thread_active
    for led in leds:
        led.value(0)
    buzzer.duty_u16(0)  # Apagar el buzzer
    IN1.value(0)
    IN2.value(0)
    IN3.value(0)
    IN4.value(0)
    melody_thread_active = False
    print("Todo apagado.")

# Función para detener la melodía
def stop_melody_thread():
    global melody_thread_active
    melody_thread_active = False
    buzzer.duty_u16(0)  # Apagar el buzzer inmediatamente

# Función para manejar mensajes MQTT
def llegada_mensaje(topic, msg):
    global estado_mqtt, melody_thread_active
    print(f"Mensaje recibido: {msg}")
    if msg == b'1':
        print("Activando por mensaje MQTT...")
        estado_mqtt = True
        for led in leds:
            led.value(1)
        if not melody_thread_active:
            _thread.start_new_thread(play_melody, ())
            melody_thread_active = True
        _thread.start_new_thread(mover_motor, (secuencia_horario, 512, 800))
    elif msg == b'0':
        print("Apagando por mensaje MQTT...")
        estado_mqtt = False
        apagar_todo()

# Función para suscribirse a MQTT
def subscribir():
    client = MQTTClient(MQTT_CLIENT_ID, MQTT_BROKER, port=MQTT_PORT, user=MQTT_USER, password=MQTT_PASSWORD, keepalive=0)
    client.set_callback(llegada_mensaje)
    client.connect()
    client.subscribe(MQTT_TOPIC)
    print(f"Conectado a {MQTT_BROKER}, en el tópico {MQTT_TOPIC}")
    return client

# Configuración del sensor
sensor = HCSR04(trigger_pin=33, echo_pin=32, echo_timeout_us=24000)

# Conectar a WiFi
conectar_wifi()
client = subscribir()

# Programa principal con monitoreo de distancia
try:
    while True:
        client.check_msg()
        distancia = sensor.distance_cm()
        print(f"Distancia: {distancia} cm")

        if not estado_mqtt:
            if distancia < 50:  # Si detecta un objeto cerca
                print("Objeto detectado: Iniciando secuencia")
                if not melody_thread_active:
                    _thread.start_new_thread(play_melody, ())
                    melody_thread_active = True
                _thread.start_new_thread(mover_motor, (secuencia_horario, 512, 800))
                for led in leds:
                    led.value(1)
                sleep(10)  # Ajusta según el tiempo total requerido
                apagar_todo()
            else:
                print("Esperando a que el objeto se acerque.")
        sleep(1)

except KeyboardInterrupt:
    stop_melody_thread()
    apagar_todo()
    print("Programa detenido.")

## Dibujo del personaje
https://drive.google.com/drive/folders/1Cucui6sjq2T2Q0s4BUArYeFyz-wNVZ3g?usp=sharing

## Enlaces de la simulación de wokwi
https://wokwi.com/projects/412573310183348225

## Capturas de Evaluaciones de Curso de JavaScript
https://drive.google.com/drive/folders/1hTDZ3OFz2C5CW8hThpVOykuvhUyovCyG?usp=drive_link

## Video de Prueba del Proyecto
https://drive.google.com/drive/folders/1L2NVgFEC1wf---T9UD5WdaxqA53_3Gxk?usp=drive_link

## Imagenes de la elaboracion del circuito
https://drive.google.com/drive/folders/1jbz5fqALngCVUj3iGY4s_2FHs4xl78DC?usp=sharing

## Elaboracion del nacimiento
https://drive.google.com/drive/folders/1TVtni9Tg3pcT1xVO0aaDITzUaDwtRRmy?usp=drive_link

## Videos de los ejercicios de clase
https://drive.google.com/drive/folders/1lpNoHPpwZ2OynrGCjJKJJWD4lfdcB9du?usp=sharing

