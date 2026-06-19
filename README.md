# Manual de Estudio Técnico y de Operación: MedalSync Telemetry

Este repositorio compendia la especificación técnica, el modelado electrónico, la configuración del entorno de red en Linux y el código fuente completo del sistema de telemetría vehicular en tiempo real desarrollado para el proyecto **MedalSync**.

---

## 1. Arquitectura General del Sistema

El proyecto implementa una arquitectura **Cliente-Servidor (Client-Server)** de tipo local e híbrida. Centraliza la recolección de métricas físicas mediante sensores directos y lógica de simulación paralela en una **Raspberry Pi**, exponiendo los datos a través de una API REST que es consumida en tiempo real por una interfaz gráfica de usuario.

[ Sensor HC-SR04 ] --(GPIO/Digital)--> [ Raspberry Pi ] <-- (Acelerómetro Virtual / Mock)
|
(Servidor Flask - Puerto 5000)
|
(Red Local / Cable Ethernet)
v
[ Laptop / Navegador Chrome ]


### Dinámica del Flujo de Datos:
1. **Muestreo:** La Raspberry Pi ejecuta un ciclo continuo de lectura digital sobre los pines GPIO asignados al sensor ultrasónico y calcula de forma paralela hilos virtuales para las variables de movimiento.
2. **Serialización (JSON):** El framework Flask empaqueta las lecturas crudas del hardware en un objeto JSON estructurado bajo el endpoint protegido `/datos`.
3. **Renderizado síncrono (Polling):** El cliente en el navegador ejecuta consultas asíncronas HTTP (`fetch`) cada 800ms, alterando la estructura del DOM de forma dinámica para reflejar los cambios al instante sin refrescar la ventana.

---

## 2. Modelado de Hardware y Teoría Eléctrica

### A. Sensor Ultrasónico (HC-SR04) - Medición de Combustible
El dispositivo realiza un cálculo indirecto de distancia miendo el tiempo de vuelo de una ráfaga de ondas mecánicas de alta frecuencia (40 kHz).

* **Funcionamiento:** Se envía un pulso de disparo (Trigger) de 10 microsegundos para activar los transductores. El pin Echo se eleva a nivel alto (5 V) y permanece así hasta que el eco de la onda acústica regresa tras chocar con la superficie del líquido.
* **Fórmula Física Aplicada:** Sabiendo que la velocidad de propagación del sonido en el aire es de aproximadamente 343 m/s (34,300 cm/s), se implementa la ecuación de movimiento rectilíneo uniforme, dividiendo entre dos debido al recorrido de ida y vuelta de la onda:

  Distancia = (Tiempo de Eco * 34,300) / 2

#### Divisor de Tensión de Seguridad:
Los pines de lectura de la Raspberry Pi toleran un máximo estricto de 3.3 V. Dado que el pin Echo entrega un pulso de salida de 5 V, se introduce un arreglo resistivo con valores de 1 kΩ (R1) y 2 kΩ (R2) en serie hacia Tierra.

La caída de tensión en el punto de medición digital se modela con la Ley de Ohm:

  Vout = Vin * ( R2 / (R1 + R2) )
  Vout = 5 V * ( 2000 Ω / (1000 Ω + 2000 Ω) ) = 3.33 V

* **TRIG:** Enlazado al pin **GPIO 23** (Pin Físico 16).
* **ECHO:** Dirigido al extremo superior de la resistencia de 1 kΩ.
* **Punto de Entrada Digital (Pi):** Conectado en el nodo central entre ambas resistencias al **GPIO 24** (Pin Físico 18).
* **GND:** Retornado a la barra de tierra de referencia común (Pin Físico 6).

### B. Acelerómetro (MPU6050) - Detección de Movimiento
* **Aislamiento por Software (Mocking):** Tras identificar cortocircuitos por falsos contactos en las láminas de la protoboard que provocaban la inundación total del bus I2C (llenando la herramienta `i2cdetect` de registros falsos), se optó por desacoplar las lecturas de los pines de datos para evitar daños permanentes en la Pi. El comportamiento tridimensional (X, Y, Z) se emula mediante un generador pseudoaleatorio de precisión en el backend.
* **Etapa de Potencia:** Se mantiene el componente energizado de manera segura conectando **VCC a 3.3V** (Pin Físico 1) y **GND** (Pin Físico 6) para validar que el circuito de filtrado y el regulador interno del sensor operan correctamente, evidenciado por el LED verde encendido.

---

## 3. Entorno de Red y Comandos de Administración (Linux)

### A. Segmentación de Red Local (Ethernet)
Al enlazar la laptop con la Raspberry Pi a través de un cable de red y compartir la conexión, se genera una subred privada local donde la Pi toma la dirección IP estática `192.168.137.88`.

* **Prueba de Conectividad (ICMP):** Para validar que el canal físico está abierto y hay comunicación bilateral antes de arrancar la aplicación, se ejecuta desde la computadora:
  ```bash
  ping 192.168.137.88
B. Liberación de Sockets Retenidos (Procesos Zombi)
Si el script de Python se interrumpe de forma abrupta mediante la combinación Ctrl + C, el núcleo de Linux puede demorar en liberar el puerto de comunicación 5000. Al intentar reiniciar la telemetría, el sistema colapsará dando un error de dirección en uso. Para matar el hilo colgado de raíz, se ejecuta en la terminal de la Pi:

Bash
sudo fuser -k 5000/tcp
4. Lógica Algorítmica del Backend
A. Calibración y Mapeo del Contenedor de 7 cm
El sensor ultrasónico presenta una zona muerta de saturación si los objetos se aproximan a menos de 2 cm de los cilindros. Para el tanque de 7 cm de profundidad, las variables se ajustan analíticamente así:

Límite de Seguridad Inferior (100% Volumen): Establecido en 2.5 cm de distancia para prevenir lecturas erráticas.

Límite Físico Superior (0% Volumen): Establecido en 7.0 cm, correspondiente al fondo del envase.

Cualquier medida intermedia se procesa mediante una interpolación matemática acotada entre 0 y 100 para evitar desbordamientos numéricos.

B. Filtro Estabilizador de Concurrencia
Para mitigar la pérdida de muestras o valores nulos (-1) provocados cuando el procesador central atiende solicitudes del servidor web y desatiende el conteo de microsegundos de los pines físicos, el algoritmo implementa un ciclo de 3 reintentos de lectura consecutivos con retardos de estabilización de 20ms antes de descartar la medición.

C. Algoritmo Analítico de Detección de Fugas
La lógica de seguridad no busca un nivel bajo fijo, sino que calcula la rapidez con la que el líquido desciende en el tiempo (derivada discreta con respecto al tiempo):

Velocidad de Vaciado = (Porcentaje Anterior - Porcentaje Actual) / Tiempo Transcurrido

Si el diferencial de volumen es negativo y la velocidad calculada es igual o mayor a una tasa de cambio del 5.0% de volumen por segundo, el backend interpreta una pérdida destructiva de fluido, activando la alerta en la respuesta JSON.

5. Código de Red del Servidor: sensores.py
Python
import time
import random
from flask import Flask, render_template, jsonify
import RPi.GPIO as GPIO

# --- PARÁMETROS DE CALIBRACIÓN DEL CONTENEDOR (7 CM) ---
DIST_MINIMA = 2.5   # Tanque al 100% (2.5 cm de distancia al sensor)
DIST_MAXIMA = 7.0   # Tanque al 0% (7.0 cm de distancia al fondo)
UMBRAL_MOVIMIENTO = 0.8  
UMBRAL_FUGA = 5.0   # Caída del 5% por segundo dispara la alerta

app = Flask(__name__)

# Configuración Inicial de Pines GPIO
TRIG = 23
ECHO = 24
GPIO.setmode(GPIO.BCM)
GPIO.setup(TRIG, GPIO.OUT)
GPIO.setup(ECHO, GPIO.IN)

# Variables globales para el análisis temporal de fuga
ultimo_porcentaje = None
ultima_lectura_tiempo = time.time()

def medir_distancia():
    """Envía un pulso de ultrasonido y calcula el tiempo de respuesta del eco."""
    GPIO.output(TRIG, False)
    time.sleep(0.1)  # Limpieza de ecos remanentes dentro del contenedor
    GPIO.output(TRIG, True)
    time.sleep(0.00001)  # Pulso de disparo de 10 microsegundos
    GPIO.output(TRIG, False)

    inicio = time.time()
    fin = time.time()
    
    # Mecanismos de protección (Timeout) contra congelamiento de hilos
    timeout = time.time() + 0.05
    while GPIO.input(ECHO) == 0:
        inicio = time.time()
        if time.time() > timeout: 
            return -1

    timeout = time.time() + 0.05
    while GPIO.input(ECHO) == 1:
        fin = time.time()
        if time.time() > timeout: 
            return -1

    duracion = fin - inicio
    return (duracion * 34300) / 2

@app.route('/')
def index():
    """Sirve la plantilla web principal al cliente."""
    return render_template('index.html')

@app.route('/datos')
def datos():
    """API Endpoint que procesa, empaqueta y retorna la telemetría en JSON."""
    global ultimo_porcentaje, ultima_lectura_tiempo
    
    datos_payload = {
        "distancia": -1,
        "porcentaje": 0,
        "acelerometro": {"x": 0.0, "y": 0.0, "z": 0.0},
        "en_movimiento": False,
        "caida_rapida": False
    }

    # Filtro tolerante a fallos (3 reintentos)
    dist = -1
    for _ in range(3):
        dist = medir_distancia()
        if dist != -1 and dist < 400.0:
            break
        time.sleep(0.02)

    # Procesamiento del volumen del contenedor
    if dist != -1:
        datos_payload["distancia"] = dist
        # Interpolación lineal matemática
        pct = (1 - (dist - DIST_MINIMA) / (DIST_MAXIMA - DIST_MINIMA)) * 100
        pct = max(0, min(100, int(pct)))
        datos_payload["porcentaje"] = pct

        # Algoritmo temporal para detección de fugas
        ahora = time.time()
        if ultimo_porcentaje is not None:
            tiempo_transcurrido = ahora - ultima_lectura_tiempo
            if tiempo_transcurrido > 0:
                velocidad_vaciado = (ultimo_porcentaje - pct) / tiempo_transcurrido
                if velocidad_vaciado >= UMBRAL_FUGA:
                    datos_payload["caida_rapida"] = True

        # Actualización de estados históricos
        if ultimo_porcentaje is None or pct <= ultimo_porcentaje or (pct - ultimo_porcentaje) > 10:
            ultimo_porcentaje = pct
            ultima_lectura_tiempo = ahora

    # --- MÓDULO MOCK: SIMULACIÓN DE SEGURIDAD DEL ACELERÓMETRO ---
    sim_x = random.uniform(-1.5, 1.5)
    sim_y = random.uniform(-1.5, 1.5)
    sim_z = random.uniform(9.0, 10.0)

    datos_payload["acelerometro"] = {"x": sim_x, "y": sim_y, "z": sim_z}
    if abs(sim_x) > UMBRAL_MOVIMIENTO or abs(sim_y) > UMBRAL_MOVIMIENTO:
        datos_payload["en_movimiento"] = True

    return jsonify(datos_payload)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, threaded=True)
6. Interfaz de Usuario Gráfica: templates/index.html
HTML
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Panel de Telemetría Vehicular</title>
    <script src="[https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4](https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4)"></script>
</head>
<body class="bg-slate-900 text-white font-sans min-h-screen p-6">

    <header class="max-w-5xl mx-auto mb-8 flex justify-between items-center border-b border-slate-700 pb-4">
        <h1 class="text-3xl font-bold tracking-tight text-blue-400">MedalSync Telemetry</h1>
        <div class="flex items-center gap-2 bg-slate-800 px-4 py-2 rounded-full border border-slate-700">
            <span class="w-3 h-3 bg-green-500 rounded-full animate-pulse"></span>
            <span class="text-sm font-semibold text-slate-300">Sistema Online</span>
        </div>
    </header>

    <main class="max-w-5xl mx-auto grid grid-cols-1 md:grid-cols-2 gap-8">
        
        <section class="bg-slate-800 p-6 rounded-2xl border border-slate-700 shadow-xl flex flex-col justify-between">
            <div>
                <h2 class="text-xl font-bold text-slate-400 mb-4 uppercase tracking-wider">Nivel de Combustible</h2>
                
                <div id="alerta-fuga" class="hidden bg-red-600/20 border-2 border-red-500 text-red-200 p-3 rounded-xl mb-4 font-bold animate-pulse text-center">
                    ⚠️ ¡ALERTA CRÍTICA: ANOMALÍA / FUGA DE COMBUSTIBLE!
                </div>

                <div class="flex items-center justify-around my-6">
                    <div class="w-24 h-56 bg-slate-950 border-4 border-slate-600 rounded-2xl relative overflow-hidden flex items-end">
                        <div id="barra-gasolina" class="w-full bg-gradient-to-t from-green-600 to-green-400 transition-all duration-500" style="height: 0%;"></div>
                    </div>
                    <div class="text-center">
                        <span id="porcentaje-texto" class="text-6xl font-black text-white block">0%</span>
                        <span id="distancia-texto" class="text-slate-400 text-sm block mt-2">Distancia: -- cm</span>
                    </div>
                </div>
            </div>
        </section>

        <section class="bg-slate-800 p-6 rounded-2xl border border-slate-700 shadow-xl flex flex-col justify-between">
            <div>
                <h2 class="text-xl font-bold text-slate-400 mb-6 uppercase tracking-wider">Estado del Vehículo</h2>
                <div class="flex justify-center items-center h-40">
                    <div id="badge-estado" class="bg-slate-950 border border-slate-700 px-8 py-6 rounded-3xl text-center shadow-lg duration-300">
                        <span id="texto-estado" class="text-3xl font-black tracking-wide text-slate-500 uppercase">Auto Detenido</span>
                    </div>
                </div>
            </div>

            <div class="grid grid-cols-3 gap-2 text-center bg-slate-900 p-3 rounded-xl border border-slate-700">
                <div><span class="text-xs text-slate-500 block">Eje X</span><span id="eje-x" class="font-mono text-sm text-blue-300">0.00</span></div>
                <div><span class="text-xs text-slate-500 block">Eje Y</span><span id="eje-y" class="font-mono text-sm text-blue-300">0.00</span></div>
                <div><span class="text-xs text-slate-500 block">Eje Z</span><span id="eje-z" class="font-mono text-sm text-blue-300">0.00</span></div>
            </div>
        </section>
    </main>

    <script>
        // Sincronización asíncrona mediante Polling Activo a 800ms
        async function actualizarDashboard() {
            try {
                const response = await fetch('/datos');
                const data = await response.json();

                // 1. Renderizado Dinámico del Nivel de Combustible
                if (data.distancia !== -1) {
                    document.getElementById('distancia-texto').innerText = `Distancia: ${data.distancia.toFixed(2)} cm`;
                    document.getElementById('porcentaje-texto').innerText = `${data.porcentaje}%`;
                    
                    const barra = document.getElementById('barra-gasolina');
                    barra.style.height = `${data.porcentaje}%`;

                    if (data.porcentaje > 50) barra.className = "w-full bg-gradient-to-t from-green-600 to-green-400";
                    else if (data.porcentaje > 20) barra.className = "w-full bg-gradient-to-t from-yellow-600 to-yellow-400";
                    else barra.className = "w-full bg-gradient-to-t from-red-600 to-red-400";
                } else {
                    document.getElementById('distancia-texto').innerText = "Error HC-SR04";
                }

                // 2. Control de la Alerta Crítica de Fuga en el DOM
                const alerta = document.getElementById('alerta-fuga');
                if (data.caida_rapida) {
                    alerta.classList.remove('hidden');
                } else {
                    alerta.classList.add('hidden');
                }

                // 3. Renderizado de Vectores Tridimensionales y Modificación de Tarjetas
                document.getElementById('eje-x').innerText = data.acelerometro.x.toFixed(2);
                document.getElementById('eje-y').innerText = data.acelerometro.y.toFixed(2);
                document.getElementById('eje-z').innerText = data.acelerometro.z.toFixed(2);

                const badge = document.getElementById('badge-estado');
                const texto = document.getElementById('texto-estado');

                if (data.en_movimiento) {
                    badge.className = "bg-green-950 border-2 border-green-500 px-8 py-6 rounded-3xl text-center shadow-2xl transition-all duration-300";
                    texto.className = "text-3xl font-black tracking-wide text-green-400 uppercase";
                    texto.innerText = "En Movimiento";
                } else {
                    badge.className = "bg-slate-950 border border-slate-700 px-8 py-6 rounded-3xl text-center shadow-lg transition-all duration-300";
                    texto.className = "text-3xl font-black tracking-wide text-slate-400 uppercase";
                    texto.innerText = "Auto Detenido";
                }

            } catch (error) {
                console.error("Fallo en la comunicación con la API de telemetría:", error);
            }
        }

        setInterval(actualizarDashboard, 800);
    </script>
</body>
</html>
7. Guía de Despliegue e Inspección en la Evaluación
Inspección Eléctrica: Verificar que las resistencias del divisor de tensión estén firmes en la protoboard y compartan la misma línea de tierra (GND).

Arranque del Backend: Ejecutar el servidor web en la terminal de la Raspberry Pi:

Bash
python3 sensores.py
Despliegue del Panel: Acceder desde Chrome en la laptop utilizando la dirección IP del puente Ethernet: http://192.168.137.88:5000

Prueba de Campo: Introducir el sensor en el contenedor de 7 cm. Al retirar el sensor bruscamente del contenedor (simulando una caída rápida de nivel), comprobar que la interfaz despliega de inmediato el banner de advertencia parpadeante.
