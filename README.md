# proyecto-sensor-alcohol - Código de implementación de un sensor de alcohol 
import machine
import time
import network
import urequests
from umqtt.simple import MQTTClient
import ujson

# Configuración del broker MQTT
BROKER = "broker.hivemq.com"
TOPIC = "vehiculo/estado"
CLIENT_ID = "sensor-alcohol"

# Configuración de la red Wi-Fi
WIFI_SSID = "Familia_Rodriguez"
WIFI_PASSWORD = "C3045821486r1997"

# Configuración del bot de Telegram
TELEGRAM_BOT_TOKEN = "7832859996:AAGvz5fzrnsT4lTPQDyrvA-5B7qML-8e-oo"
TELEGRAM_CHAT_ID = "6138888116"

# Configuración de pines
alcohol_sensor = machine.ADC(machine.Pin(34))  # MQ-3 en pin D2 (GPIO4)
#alcohol_sensor.atten(machine.ADC.ATTN_11DB)
buzzer_led_pin = machine.Pin(5, machine.Pin.OUT)  # Buzzer y LED rojo en pin D5 (GPIO5)
led_green = machine.Pin(18, machine.Pin.OUT)  # LED verde en pin D4 (GPIO2)

# Umbrales y configuración de histéresis
ALCOHOL_THRESHOLD = 900
HYSTERESIS_MARGIN = 200  # Margen para evitar fluctuaciones
ALARM_DURATION = 5  # Duración de la alarma (segundos)

# Configuración del filtro de promedio móvil
READINGS_COUNT = 10  # Cantidad de lecturas para promediar
readings = [0] * READINGS_COUNT  # Lista para almacenar lecturas

# Calentamiento del sensor (en segundos)
SENSOR_WARMUP_TIME = 5  # Tiempo recomendado para estabilización

# Estado del sistema
is_blocked = False  # Estado del vehículo (bloqueado o desbloqueado)
MANUAL_UNLOCK = False  # Desbloqueo manual desde Telegram
UNLOCK_AVAILABLE = True  # Disponibilidad del comando /desbloquear

# Conexión Wi-Fi
def connect_to_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASSWORD)
    print("Conectando a Wi-Fi...")
    while not wlan.isconnected():
        time.sleep(1)
    print("Conectado a Wi-Fi")
    print("Dirección IP:", wlan.ifconfig()[0])

# Enviar datos a MQTT
def send_mqtt_message(client, topic, message):
    try:
        client.connect()
        client.publish(topic, message)
        client.disconnect()
    except Exception as e:
        print("Error al enviar mensaje MQTT:", e)

# Enviar notificación a Telegram
def send_telegram_message(message):
    try:
        url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
        data = {"chat_id": TELEGRAM_CHAT_ID, "text": message}
        response = urequests.post(url, json=data)
        response.close()
    except Exception as e:
        print("Error al enviar mensaje a Telegram:", e)

# Escuchar comandos de Telegram
def check_telegram_updates():
    global MANUAL_UNLOCK, UNLOCK_AVAILABLE
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/getUpdates"
    try:
        response = urequests.get(url)
        updates = response.json().get("result", [])
        for update in updates:
            if "message" in update and "text" in update["message"]:
                text = update["message"]["text"]
                if text.lower() == "/desbloquear" and UNLOCK_AVAILABLE:
                    MANUAL_UNLOCK = True
                    UNLOCK_AVAILABLE = False
                    send_telegram_message("Vehículo desbloqueado manualmente. Comando deshabilitado hasta el próximo bloqueo.")
        response.close()
    except Exception as e:
        print("Error al verificar actualizaciones de Telegram:", e)

# Filtrar lecturas usando promedio móvil
def get_filtered_reading(raw_value):
    global readings
    readings.pop(0)  # Eliminar la lectura más antigua
    readings.append(raw_value)  # Añadir la nueva lectura
    return sum(readings) // len(readings)  # Promedio de las lecturas

# Inicializar cliente MQTT
client = MQTTClient(CLIENT_ID, BROKER)

# Conexión Wi-Fi
connect_to_wifi()

# Calentamiento del sensor
print(f"Calentando el sensor MQ-3 durante {SENSOR_WARMUP_TIME} segundos...")
time.sleep(SENSOR_WARMUP_TIME)
print("Sensor MQ-3 listo para operar.")
led_green.value(1)

# Inicialización de lecturas
readings = [alcohol_sensor.read()] * READINGS_COUNT

# LED verde para indicar encendido
led_green.value(1)
print("Sistema iniciado")

while True:
    # Leer el valor crudo del sensor
    raw_alcohol_level = alcohol_sensor.read()

    # Ignorar lecturas inusualmente bajas
    if raw_alcohol_level < 100:  # Umbral mínimo
        raw_alcohol_level = 0

    # Aplicar el filtro de promedio móvil
    alcohol_level = get_filtered_reading(raw_alcohol_level)

    print("Nivel de alcohol (crudo):", raw_alcohol_level)
    print("Nivel de alcohol (filtrado):", alcohol_level)

    # Determinar estado del vehículo usando histéresis
    if not is_blocked and alcohol_level > ALCOHOL_THRESHOLD + HYSTERESIS_MARGIN:
        is_blocked = True
        MANUAL_UNLOCK = False  # Deshabilitar desbloqueo manual si se detecta alcohol
        estado = "Bloqueado"
        embriaguez = "Sí"
        buzzer_led_pin.value(1)  # Activar LED rojo y buzzer
        print("Estado de embriaguez detectado. Vehículo bloqueado.")
        send_telegram_message(f"Estado de embriaguez detectado.\nNivel: {alcohol_level}.\nVehículo bloqueado.")
        time.sleep(ALARM_DURATION)
        buzzer_led_pin.value(0)  # Apagar alarma después de 10 segundos
        led_green.value(0)
    elif is_blocked and raw_alcohol_level < 980:
        is_blocked = False
        estado = "Desbloqueado"
        embriaguez = "No"
        print("Nivel de alcohol aceptable. Vehículo desbloqueado.")
        led_green.value(1)
    else:
        estado = "Bloqueado" if is_blocked else "Desbloqueado"
        embriaguez = "Sí" if is_blocked else "No"

    # Enviar datos a MQTT
    message = ujson.dumps({
        "nivel": alcohol_level,
       # "estado": estado,
       # "embriaguez": embriaguez
    })
    send_mqtt_message(client, TOPIC, message)

    # Verificar comandos de Telegram
    check_telegram_updates()

    time.sleep(2)
