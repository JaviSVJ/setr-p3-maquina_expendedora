# Práctica 3: Controlador de Máquina Expendedora en Arduino (SETR)

## 1. Descripción General de la Solución
El objetivo de esta práctica es implementar el firmware de control de una máquina expendedora sobre Arduino Uno, empleando una arquitectura reactiva sin bloqueos ni espera actiba, respetando los tiempos de interacción, modos de operación y restricciones hardware.

El sistema se organiza como una **Máquina de Estados Jerárquica** dividida en tres modos principales:
1. **Modo Arranque (`MODO_ARRANQUE`):** Muestra `"CARGANDO"` en el LCD 16x2 mientras el **LED1 (Rojo)** parpadea 3 veces a una frecuencia exacta de 1 Hz (3 segundos encendido y 3 segundos apagado alternativamente).
2. **Modo Servicio (`MODO_SERVICIO`):**
   * *Espera (`SRV_ESPERANDO`):* Muestra `"ESPERANDO CLIENTE"` hasta que el sensor de ultrasonidos detecta presencia a menos de $1\text{ m}$ ($100\text{ cm}$).
   * *Entorno (`SRV_ENTORNO`):* Muestra la temperatura y humedad ambiente medidas por el DHT11 durante $5\text{ segundos}$.
   * *Catálogo (`SRV_CATALOGO`):* Permite navegar de forma circular con el eje Y del joystick entre 5 productos (`Cafe Solo`, `Cafe Cortado`, `Cafe Doble`, `Cafe Premium`, `Chocolate`) y seleccionarlo pulsando el botón central (`SW`).
   * *Preparación (`SRV_PREPARANDO`):* Muestra `"Preparando Cafe..."` durante un tiempo pseudoaleatorio entre $4$ y $8\text{ segundos}$ (`random(4000, 8001)`), incrementando de forma proporcional la luminosidad del **LED2 (Verde)** mediante **PWM**.
   * *Retirada (`SRV_RETIRAR`):* Apaga el LED2, muestra `"RETIRE BEBIDA"` durante $3\text{ segundos}$ y regresa al estado de espera.
   * *Cancelación:* Si en cualquier punto del Modo Servicio se mantiene pulsado el botón externo entre $2$ y $3\text{ segundos}$, se cancela la operación actual y se vuelve al inicio del Modo Servicio.
3. **Modo Administración (`MODO_ADMIN`):** Se entra y sale manteniendo pulsado el botón externo durante $\ge 5\text{ segundos}$ consecutivos. En este modo permanecen encendidos tanto **LED1** como **LED2**, y el joystick permite consultar temperatura/humedad, distancia actual, contador de segundos desde el arranque y modificar en tiempo real el precio de los productos (en pasos de $\pm 0.05\,\text{EUR}$, confirmando con el pulsador del joystick o cancelando hacia la izquierda).

## 2. Recursos de Tiempo Real y Decisiones de Diseño

Para garantizar que la interfaz responda al instante sin perder pulsaciones ni bloquear sensores, únicamente se ha utilizado delay para los pulsos del sensor de ultrasonidos.
Para sustituir el uso de delay en el codigo y evitar asi la espera activa he utilizado : 

### A. Multitarea Cooperativa con `ArduinoThread`
Se emplea `ThreadController` para planificar 4 hilos periódicos independientes en `loop()`:
* **`threadUltrasonidos` (cada $150\text{ ms}$):** Dispara el HC-SR04 y mide el eco mediante `pulseIn(PIN_ECHO, HIGH, 25000)`. Se fija un *timeout* de $25\,000\,\mu\text{s}$ ($\approx 4.3\text{ m}$) para evitar que la función bloquee el sistema durante $1\text{ segundo}$ si no recibe rebote.
* **`threadDHT` (cada $2000\text{ ms}$):** Lee temperatura y humedad del sensor DHT11 respetando su periodo mínimo de muestreo de $2\text{ segundos}$ y descartando lecturas erróneas (`isnan()`).
* **`threadJoystick` (cada $80\text{ ms}$):** Lee los canales analógicos `A0`/`A1` con histéresis (zona muerta central entre `350` y `670` para evitar desplazamientos múltiples por una sola inclinación) y detecta el flanco de pulsación del botón integrado `SW`.
* **`threadControl` (cada $100\text{ ms}$):** Ejecuta las transiciones de la máquina de estados, regula el PWM de `LED2` y actualiza el LCD únicamente cuando cambia el contenido de alguna línea, evitando parpadeos visuales.

### B. Interrupciones Hardware (`attachInterrupt`) y Software (`TimerOne`)
* **Interrupción Hardware en Pin 2 (`INT0`, modo `CHANGE`):** El pulsador externo está conectado entre `D2` y `GND` usando `INPUT_PULLUP`. La rutina `isrBotonExterno()` incorpora un filtro antirrebote por software de $50\text{ ms}$ con `millis()` y mide con exactitud el instante de bajada y subida para distinguir la pulsación de $2\text{–}3\text{ s}$ (cancelar) de la pulsación continua $\ge 5\text{ s}$ (Modo Admin).
* **Interrupción periódica por Timer (`TimerOne` a $1\text{ Hz}$):** Se configura `Timer1.initialize(1000000)` para generar una base de tiempos exacta de $1\text{ segundo}$. Su ISR (`isrTimer1()`) incrementa el contador global de tiempo activo y conmuta `LED1` durante los $6$ semiperiodos ($3$ parpadeos) del arranque.
* **Evitación de conflictos entre Timers y PWM:** Dado que la librería `TimerOne` utiliza el temporizador hardware **Timer1** (que controla el PWM de los pines `9` y `10`), el **LED2** se ha conectado al **Pin 5**, gestionado por **Timer0**, permitiendo variar su brillo con `analogWrite()` sin interferencias.
* **Variables `volatile` y Secciones Críticas:** Todas las variables compartidas entre las rutinas de interrupción y `tareaControl()` llevan el calificador `volatile` para evitar optimizaciones en registros del compilador, y se copian de forma atómica dentro de bloques `noInterrupts()` / `interrupts()`.

### C. Tolerancia a Fallos con Watchdog Timer (`avr/wdt.h`)
Se habilita el perro guardián hardware con un margen de $4\text{ segundos}$ (`wdt_enable(WDTO_4S)`) al finalizar `setup()`, reiniciando su contador en cada iteración de `loop()` mediante `wdt_reset()`. Si algún sensor o bucle quedara bloqueado más de $4\text{ segundos}$, el microcontrolador se reinicia automáticamente de forma segura.

## 3. ESQUEMA EN FRITZING Y CIRCUITO REAL

<img width="1186" height="740" alt="image" src="https://github.com/user-attachments/assets/e4754fb2-4b9d-4ed3-ae96-fa5f8fbe2458" />


<img width="2000" height="924" alt="image" src="https://github.com/user-attachments/assets/2a3cf255-1dda-44b5-b9c0-43f81c7d2568" />

