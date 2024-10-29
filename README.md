# Practica Integral 2
Proyecto de IoT

## Descripción
Este proyecto utiliza un sensor de ultrasonido HCSR04 para medir la distancia y controlar un buzzer y un LED RGB. El sistema detecta la proximidad de un objeto y responde encendiendo diferentes colores de LED y reproduciendo melodías en el buzzer. Un botón se utiliza para reiniciar el estado del sistema.

## Código del Programa

```python
from machine import Pin, PWM
from hcsr04 import HCSR04
import time

# Configuración del sensor HCSR04
sensor = HCSR04(trigger_pin=15, echo_pin=4, echo_timeout_us=24000)

# Configuración de pines
boton = Pin(14, Pin.IN, Pin.PULL_UP)
buzzer = PWM(Pin(27))  # Pin del buzzer
led_rojo = Pin(12, Pin.OUT)
led_verde = Pin(2, Pin.OUT)
led_azul = Pin(13, Pin.OUT)

# Variables de estado
estado_inicial = True
boton_presionado = False

# Funciones para manejar los colores del LED RGB
def encender_led_verde():
    led_rojo.off()
    led_verde.on()
    led_azul.off()

def encender_led_amarillo():
    led_rojo.on()
    led_verde.on()
    led_azul.off()

def encender_led_rojo():
    led_rojo.on()
    led_verde.off()
    led_azul.off()

# Función para activar el buzzer
def activar_buzzer():
    buzzer.freq(1000)
    buzzer.duty(800)  # Volumen del buzzer (ajustable)

# Función para desactivar el buzzer
def desactivar_buzzer():
    buzzer.duty(0)

# Función para tocar una melodía en el buzzer
def reproducir_melodia():
    notas = [262, 294, 330, 349, 392, 440, 494, 523]  # Frecuencias en Hz para una escala básica
    duracion = 0.2  # Duración de cada nota en segundos
    for nota in notas:
        buzzer.freq(nota)
        buzzer.duty(800)  # Volumen del buzzer (ajustable)
        time.sleep(duracion)
    desactivar_buzzer()  # Apagar el buzzer al terminar la melodía

# Programa principal
while True:
    # Detectar proximidad y cambiar estados
    distancia = sensor.distance_cm()
    
    # Mensaje de depuración para la distancia
    print(f"Distancia detectada: {distancia} cm")

    if distancia > 20:  # Lejos
        encender_led_verde()
        desactivar_buzzer()
    elif 10 < distancia <= 20:  # Distancia media
        encender_led_amarillo()
        desactivar_buzzer()
    elif 5 < distancia <= 10:  # Cercano, solo encender LED rojo
        encender_led_rojo()
        desactivar_buzzer()  # Sin sonido
    elif distancia <= 5:  # Muy cercano
        encender_led_rojo()
        reproducir_melodia()  # Reproducir una melodía en el buzzer

    # Comprobar el estado del botón
    if boton.value() == 0 and not boton_presionado:  # Si el botón se presiona
        boton_presionado = True
        encender_led_verde()  # Cambia el LED a verde
        desactivar_buzzer()  # Apaga el buzzer
        print("Botón presionado. LED verde encendido.")  # Mensaje de depuración
        time.sleep(0.5) 
    if boton.value() == 1:  # Resetea el estado al soltar el botón
        boton_presionado = False

    time.sleep(0.5)  # Retardo para evitar rebotes

#Inicio de la libreria
from machine import Pin, time_pulse_us
from utime import sleep_us

_version_ = '0.2.1'
_author_ = 'Roberto Sánchez'
_license_ = "Apache License 2.0. https://www.apache.org/licenses/LICENSE-2.0"

class HCSR04:
    """
    Driver to use the ultrasonic sensor HC-SR04.
    The sensor range is between 2cm and 4m.

    The timeouts received listening to echo pin are converted to OSError('Out of range')
    """
    def __init__(self, trigger_pin, echo_pin, echo_timeout_us=500*2*30):
        """
        trigger_pin: Output pin to send pulses
        echo_pin: Readonly pin to measure the distance. The pin should be protected with 1k resistor
        echo_timeout_us: Timeout in microseconds to listen to echo pin. 
        By default, it is based on the sensor limit range (4m)
        """
        self.echo_timeout_us = echo_timeout_us
        # Init trigger pin (out)
        self.trigger = Pin(trigger_pin, mode=Pin.OUT, pull=None)
        self.trigger.value(0)

        # Init echo pin (in)
        self.echo = Pin(echo_pin, mode=Pin.IN, pull=None)

    def _send_pulse_and_wait(self):
        """
        Send the pulse to trigger and listen on echo pin.
        We use the method machine.time_pulse_us() to get the microseconds until the echo is received.
        """
        self.trigger.value(0)  # Stabilize the sensor
        sleep_us(5)
        self.trigger.value(1)
        # Send a 10us pulse.
        sleep_us(10)
        self.trigger.value(0)
        try:
            pulse_time = time_pulse_us(self.echo, 1, self.echo_timeout_us)
            if pulse_time < 0:
                MAX_RANGE_IN_CM = 500  # it's really ~400 but I've read people say it works up to ~460
                pulse_time = int(MAX_RANGE_IN_CM * 29.1)  # 1cm each 29.1us
            return pulse_time
        except OSError as ex:
            if ex.args[0] == 110:  # 110 = ETIMEDOUT
                raise OSError('Out of range')
            raise ex

    def distance_mm(self):
        """
        Get the distance in millimeters without floating-point operations.
        """
        pulse_time = self._send_pulse_and_wait()
        mm = pulse_time * 100 // 582
        return mm

    def distance_cm(self):
        """
        Get the distance in centimeters with floating-point operations.
        It returns a float.
        """
        pulse_time = self._send_pulse_and_wait()
        cms = (pulse_time / 2) / 29.1
        return cms

###
