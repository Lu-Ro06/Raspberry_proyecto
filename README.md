Markdown# Manual de Estudio Técnico y de Operación
## Sistema de Telemetría Vehicular en Tiempo Real (MedalSync)

---

## 1. Arquitectura General del Sistema
El proyecto es un sistema de telemetría embebido híbrido (físico/simulado) que recopila datos de sensores en una **Raspberry Pi** y los transmite a una interfaz web en tiempo real a través de una arquitectura **Client-Server** local basada en **HTTP/JSON**.

[ Sensor HC-SR04 ] --(GPIO/Digital)--> [ Raspberry Pi ] <-- (Lógica Mock por Software)|(Servidor Flask - Puerto 5000)|(Red Local / Cable)v[ Laptop / Navegador Chrome ]
---

## 2. Diagramas y Especificaciones de Hardware

### A. Sensor Ultrasónico (HC-SR04) - Medición de Combustible
* **Principio de Funcionamiento:** Emite una ráfaga de alta frecuencia (40 kHz) y mide el tiempo que tarda el eco en regresar tras chocar con la superficie del agua.
* **Cálculo de Distancia:** La velocidad del sonido es de $\approx 343\text{ m/s}$ ($34,300\text{ cm/s}$). La distancia se calcula con la fórmula:
$$\text{Distancia} = \frac{\text{Tiempo de Eco} \times 34,300}{2}$$

#### Conexión Física y Divisor de Tensión:
Dado que el pin **ECHO** del sensor emite $5\text{ V}$ y los pines GPIO de la Raspberry Pi operan a un máximo de $3.3\text{ V}$, se implementa un **divisor de tensión** con resistencias de $1\text{ k}\Omega$ ($R_1$) y $2\text{ k}\Omega$ ($R_2$) para proteger la integridad de la placa.

* **VCC:** Conectado al **Pin Físico 2 o 4** ($5\text{ V}$).
* **TRIG:** Conectado directo al **GPIO 23** (**Pin Físico 16**).
* **ECHO:** Va a la fila de la protoboard donde inicia la resistencia de $1\text{ k}\Omega$.
* **Punto de Lectura (GPIO 24 / Pin Físico 18):** Se conecta exactamente en el nodo central donde se unen la resistencia de $1\text{ k}\Omega$ y la de $2\text{ k}\Omega$.
* **GND:** La pata libre de la resistencia de $2\text{ k}\Omega$ y el pin GND del sensor van a la línea de tierra común (**Pin Físico 6, 9 o 14**).

### B. Sensor de Movimiento (MPU6050) - Estado del Vehículo
* **Aviso de simulación:** Tras experimentar fallas críticas en el bus físico de datos I2C (donde un cortocuito inundaba todas las direcciones lógicas), el hardware se desacopló y se sustituyó por un **Mock virtual por software** de alta fidelidad, asegurando la continuidad operativa del sistema.
* **Conexión de Respaldo Física (Solo alimentación para validación visual de energía):**
  * **VCC:** Conectado al **Pin Físico 1** ($3.3\text{ V}$).
  * **GND:** Conectado a la línea de tierra común (**Pin Físico 6**).
  * *Nota: El LED verde encendido valida que la etapa de potencia del sensor se encuentra energizada de manera correcta.*

---

## 3. Configuración del Entorno de Red y Sistema (Linux/Ubuntu)

Para que la laptop pueda visualizar la telemetría sin depender de una red Wi-Fi externa, se utiliza un puente de red por cable Ethernet mediante el protocolo **ICMP/IP**.

### A. Comprobación de Conectividad (Desde la Laptop)
Para verificar la visibilidad directa hacia la Raspberry Pi a través del segmento compartido `192.168.137.xx`, se ejecuta en la consola de Windows:
```cmd
ping 192.168.137.88
B. Administración de Puertos Bloqueados en LinuxCuando el servidor de Flask se cierra de forma abrupta (Ctrl + C), el puerto lógico de comunicación puede quedar retenido en estado zombi. Para liberar por la fuerza el socket TCP y solucionar el error ERR_CONNECTION_REFUSED, se ejecuta:Bashsudo fuser -k 5000/tcp
4. Guía de Despliegue Paso a Paso (Checklist para el Examen)Verificar Conexiones: Valida físicamente que las resistencias de $1\text{ k}\Omega$ y $2\text{ k}\Omega$ hagan puente limpio en la protoboard y que el cable de lectura vaya al Pin Físico 18.Encendido del Servidor: Ejecuta en la consola de la Raspberry Pi: python3 sensores.pyAcceso Remoto: Abre Google Chrome en tu laptop e ingresa a la URL: http://192.168.137.88:5000Prueba de Validación de Combustible: Introduce el sensor ultrasónico en tu bote de 7 cm. Al simular un vaciado rápido sacando el sensor del bote de golpe, la interfaz web activará instantáneamente la Alerta Roja Intermitente de Fuga.
---

### 2. Servidor de Red: `sensores.py`
Ejecuta: `nano sensores.py` y pega el código backend robusto calibrado a tus 7 cm:

```python
import time
import random  # Utilizado para el módulo de simulación del acelerómetro
from flask import Flask, render_template, jsonify
import RPi.GPIO as GPIO

# --- CONFIGURACIÓN DE PARÁMETROS DEL CONTENEDOR (CALIBRADO A 7 CM) ---
DIST_MINIMA = 2.5   # cm cuando el líquido está al máximo (100% Lleno)
DIST_MAXIMA = 7.0   # cm cuando el tanque está vacío en el fondo (0% Vacío)
UMBRAL_MOVIMIENTO = 0.8  # Límite de aceleración para considerar que el auto avanza
UMBRAL_FUGA = 5.0   # % de caída por segundo requerido para disparar la alerta

app = Flask(__name__)

# Configuración inicial de Pines de la Raspberry Pi
TRIG = 23
ECHO = 24
GPIO.setmode(GPIO.BCM)  # Modo de direccionamiento por canal broadcom
GPIO.setup(TRIG, GPIO.OUT)
GPIO.setup(ECHO, GPIO.IN)

# Variables globales de estado para el cálculo analítico de fuga
ultimo_porcentaje = None
ultima_lectura_tiempo = time.time()

def medir_distancia():
    """Envía un pulso ultrasónico y calcula el tiempo de respuesta del eco."""
    GPIO.output(TRIG, False)
    time.sleep(0.1)  # Ventana de tiempo esencial para limpiar ecos remanentes en 7cm
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
    
    # Estructura base del Payload JSON
    datos_payload = {
        "distancia": -1,
        "porcentaje": 0,
        "acelerometro": {"x": 0.0, "y": 0.0, "z": 0.0},
        "en_movimiento": False,
        "caida_rapida": False
    }

    # Filtro de tolerancia a fallos por ráfagas de red (3 Reintentos)
    dist = -1
    for _ in range(3):
        dist = medir_distancia()
        if dist != -1 and dist < 400.0:
            break
        time.sleep(0.02)

    # Procesamiento de telemetría del tanque de gasolina
    if dist != -1:
        datos_payload["distancia"] = dist
        # Interpolación lineal para mapear centímetros a porcentaje de volumen
        pct = (1 - (dist - DIST_MINIMA) / (DIST_MAXIMA - DIST_MINIMA)) * 100
        pct = max(0, min(100, int(pct)))
        datos_payload["porcentaje"] = pct

        # Algoritmo matemático para detección de fugas (Derivada con respecto al tiempo)
        ahora = time.time()
        if ultimo_porcentaje is not None:
            tiempo_transcurrido =超 = ahora - ultima_lectura_tiempo
            if tiempo_transcurrido > 0:
                # Calcula la tasa de cambio porcentual por segundo
                velocidad_vaciado = (ultimo_porcentaje - pct) / tiempo_transcurrido
                if velocidad_vaciado >= UMBRAL_FUGA:
                    datos_payload["caida_rapida"] = True

        # Actualización segura del histórico de volumen
        if ultimo_porcentaje is None or pct <= ultimo_porcentaje or (pct - ultimo_porcentaje) > 10:
            ultimo_porcentaje = pct
            ultima_lectura_tiempo = ahora

    # --- ENTORNO MOCK: SIMULACIÓN DE SEGURIDAD DEL ACELERÓMETRO ---
    sim_x = random.uniform(-1.5, 1.5)
    sim_y = random.uniform(-1.5, 1.5)
    sim_z = random.uniform(9.0, 10.0)

    datos_payload["acelerometro"] = {"x": sim_x, "y": sim_y, "z": sim_z}
    if abs(sim_x) > UMBRAL_MOVIMIENTO or abs(sim_y) > UMBRAL_MOVIMIENTO:
        datos_payload["en_movimiento"] = True

    return jsonify(datos_payload)

if __name__ == '__main__':
    # Lanzamiento del servidor escuchando en todas las interfaces de red (0.0.0.0)
    app.run(host='0.0.0.0', port=5000, threaded=True)
3. Interfaz Gráfica: templates/index.htmlCrea la carpeta de vistas si no existe (mkdir -p templates), ejecuta nano templates/index.html y pega la interfaz completa:HTML<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Panel de Telemetría Vehicular</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
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
                    <div id="badge-estado" class="bg-slate-950 border border-slate-700 px-8 py-6 rounded-3xl text-center shadow-lg transition-all duration-300">
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
        // Polling activo a 800ms
        async function actualizarDashboard() {
            try {
                const response = await fetch('/datos');
                const data = await response.json();

                // 1. Renderizado de Combustible e indicador dinámico de color
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

                // 2. Control de visibilidad del banner de fuga (Estrategia DOM de clases)
                const alerta = document.getElementById('alerta-fuga');
                if (data.caida_rapida) {
                    alerta.classList.remove('hidden');
                } else {
                    alerta.classList.add('hidden');
                }

                // 3. Renderizado de vectores de movimiento y alteración de estilos del badge
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
                console.error("Error en la sincronización del fetch:", error);
            }
        }

        setInterval(actualizarDashboard, 800);
    </script>
</body>
</html>
