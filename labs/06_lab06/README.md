# Proyecto Final -- Sistema Integrado: Semáforo + LCD 16×2
**ECCI · Técnicas Digitales · Verilog / MAX10**

---

## 1. Descripción General

Este proyecto integra dos módulos digitales sobre una única FPGA MAX10 (DE10-Lite), consolidando los conceptos trabajados durante el semestre: máquinas de estado finito, lógica secuencial y comunicación con periféricos.

El semáforo opera con su ciclo fijo. La LCD 16×2 actúa como panel de información: muestra en tiempo real el estado actual del semáforo y los segundos que quedan antes de que cambie.

| Módulo | Función | Salida física |
|--------|---------|---------------|
| **Semáforo** | FSM Moore — (3 a 4) estados, ciclo fijo | LEDs |
| **Controlador LCD** | Inicializa y actualiza la LCD HD44780 | Pantalla  |

Ambos módulos son instanciados desde un `top.v` que los conecta y comparte el reloj de 50 MHz.

---

## 2. Objetivos

Al finalizar este proyecto, el estudiante será capaz de:

- Integrar múltiples módulos Verilog en un diseño jerárquico con un `top-level`.
- Comprender y aplicar el protocolo de inicialización HD44780 en modo 4 bits.
- Exponer señales internas de una FSM existente para comunicarlas a otro módulo.
- Diseñar una FSM de controlador LCD con fases de inicialización, escritura y actualización.
- Sintetizar, implementar y programar el diseño completo en la FPGA.

---

## 3. Arquitectura del Sistema

```
                   ┌──────────────────────────────────────────┐
                   │               TOP LEVEL                  │
                   │                                          │
clk_50Mhz  ───────▶│  ┌────────────────┐                      │──▶ led_green
rst        ───────▶│  │   Semáforo     │──── state[1:0] ─┐    │──▶ led_yellow
                   │  │  (FSM Moore)   │──── cnt[4:0]   ─┤    │──▶ led_red
                   │  └────────────────┘                  │   │
                   │                                      ▼   │
                   │  ┌────────────────┐   ┌──────────────┐   │──▶ lcd_rs
                   │  │  clk_divider   │──▶│  lcd_ctrl    │   │──▶ lcd_e
                   │  │  100 MHz→ 1Hz  │   │ (FSM HD44780)│   │──▶ lcd_d[7:4]
                   │  └────────────────┘   └──────────────┘   │
                   └──────────────────────────────────────────┘
```

### 3.1 Contenido de la LCD

```
┌────────────────┐
│SEMAFORO: VERDE │   ← Línea 1: estado actual
│TIEMPO:    10 s │   ← Línea 2: segundos restantes
└────────────────┘
```

| Estado FSM | Línea 1 | Línea 2 |
|------------|---------|---------|
| `S_VERDE`    | `SEMAFORO: VERDE ` | `TIEMPO:    XX s ` |
| `S_AMARILLO` | `SEMAFORO: AMARIL` | `TIEMPO:    XX s ` |
| `S_ROJO`     | `SEMAFORO: ROJO  ` | `TIEMPO:    XX s ` |

El tiempo restante se calcula en el controlador LCD como:

```
tiempo_restante = DURACION_ESTADO - cnt
```

Cada vez que `state` cambia, el controlador detecta la transición y lanza la secuencia de actualización.

---

## 4. Módulos Verilog

### 4.1 `clk_divider.v` — Divisor de reloj *(reutilizado del Lab 05)*

Sin modificaciones. Genera 1 Hz a partir de los 50 MHz del oscilador de la DE10-Lite.

### 4.2 `traffic_light.v` — Semáforo FSM *(modificado del Lab 05)*

La FSM del Lab 05 debe modificarse para **exponer dos señales** que antes eran internas:

| Puerto nuevo | Tipo | Descripción |
|---|---|---|
| `state [1:0]` | `output` | Estado actual: `00`=VERDE, `01`=AMARILLO, `10`=ROJO |
| `cnt [4:0]`   | `output` | Segundos transcurridos en el estado actual |

Con estas dos señales, el controlador LCD puede mostrar el tiempo restante sin necesidad de conocer la lógica interna del semáforo.

### 4.3 `lcd_ctrl.v` — Controlador LCD *(nuevo — módulo central del proyecto)*

Recibe `state` y `cnt` del semáforo e implementa el protocolo HD44780 en modo 4 bits. Su FSM tiene tres fases:

1. **Inicialización** — envía la secuencia de comandos al encender.
2. **Escritura** — escribe las dos líneas por primera vez.
3. **Actualización** — monitorea `state`; al detectar un cambio, reescribe la pantalla.

### 4.4 `top.v` — Módulo top-level *(nuevo)*

Instancia y conecta los tres módulos anteriores. Es el único archivo que referencia los pines físicos en el `.ucf`.

---

## 5. Protocolo HD44780 — Inicialización en 4 bits

El bus de datos usa los pines **D4–D7** de la LCD (los 4 bits bajos D0–D3 se dejan sin conectar). Cada byte se envía en **dos nibbles**: primero el nibble **alto** (bits 7–4), luego el nibble **bajo** (bits 3–0), con un pulso de Enable entre cada uno.

### 5.1 Mapeo de pines

```
FPGA          LCD
lcd_d[4]  →  D4
lcd_d[5]  →  D5
lcd_d[6]  →  D6
lcd_d[7]  →  D7
               D0–D3: sin conectar
```

En Verilog, el nibble alto de un byte `data[7:0]` se coloca así:

```verilog
lcd_d <= data[7:4];   // nibble alto → D7–D4
// pulso E
lcd_d <= data[3:0];   // nibble bajo → D7–D4 (reusando el mismo bus)
// pulso E
```

> ⚠️ Aunque el bus físico son D4–D7, en Verilog se declara como `lcd_d[7:4]` y se asigna directamente el nibble correspondiente del byte. **No se hace corrimiento de bits.**

### 5.2 Secuencia de inicialización

```
Power ON
│
└─▶ Esperar > 40 ms
    │
    ├─▶ D7–D4 = 0x3, pulso E  →  esperar > 4.1 ms   (wake-up 1)
    ├─▶ D7–D4 = 0x3, pulso E  →  esperar > 100 µs   (wake-up 2)
    ├─▶ D7–D4 = 0x3, pulso E  →  esperar > 100 µs   (wake-up 3)
    ├─▶ D7–D4 = 0x2, pulso E  →  esperar > 100 µs   (cambiar a 4 bits)
    │
    │   A partir de aquí cada byte = nibble alto + nibble bajo
    │
    ├─▶ 0x28 → nibble 0x2, pulso E / nibble 0x8, pulso E   (Function Set: 4 bits, 2 líneas, 5×8)
    ├─▶ 0x08 → nibble 0x0, pulso E / nibble 0x8, pulso E   (Display OFF)
    ├─▶ 0x01 → nibble 0x0, pulso E / nibble 0x1, pulso E   →  esperar > 1.52 ms  (Clear Display)
    ├─▶ 0x06 → nibble 0x0, pulso E / nibble 0x6, pulso E   (Entry Mode: incremento, sin shift)
    └─▶ 0x0C → nibble 0x0, pulso E / nibble 0xC, pulso E   (Display ON, cursor OFF, blink OFF)
        │
        └─▶ Listo para escritura
```

### 5.3 Envío de un byte en modo 4 bits

```
rs = 0 (comando) o 1 (dato ASCII)
│
├─▶ lcd_d[7:4] = byte[7:4]   (nibble alto)
├─▶ lcd_e = 1  →  esperar ≥ 230 ns
├─▶ lcd_e = 0  →  esperar ≥ 500 ns
│
├─▶ lcd_d[7:4] = byte[3:0]   (nibble bajo en el mismo bus)
├─▶ lcd_e = 1  →  esperar ≥ 230 ns
└─▶ lcd_e = 0  →  esperar ≥ 500 ns (+ tiempo de ejecución del comando)
```


Para **escribir un carácter** ASCII: `lcd_rs = 1`, enviar los dos nibbles del código ASCII.  
Para **enviar un comando**: igual pero con `lcd_rs = 0`.


---

## 6. Procedimiento

### Paso 1 — Modificar `traffic_light.v`

Abra el código del Lab 05 y agregue `state` y `cnt` como puertos de salida. Verifique que `cnt` se reinicia a cero cada vez que la FSM cambia de estado.

> **Punto de verificación 1:** Simule el semáforo modificado en Testbench. Muestre al docente que `state` y `cnt` se comportan correctamente antes de continuar.

### Paso 2 — Diseñar la FSM de `lcd_ctrl.v`

Antes de escribir código, dibuje en papel el **diagrama de estados completo** del controlador LCD. Identifique claramente:

- Un estado por cada comando de la secuencia de inicialización.
- Los estados de escritura de línea 1 y línea 2 (puede usar un contador de caracteres).
- El estado `IDLE` y la condición de transición hacia `UPDATE`.

> **Punto de verificación 2:** Muestre el diagrama al docente y obtenga aprobación antes de implementar.

### Paso 3 — Implementar `lcd_ctrl.v`

Con el diagrama aprobado, implemente la FSM en Verilog usando la **estructura de 3 bloques**:

```
always @(posedge clk)  →  registro de estado + timer
always @(*)            →  lógica de siguiente estado
always @(*)            →  lógica de salidas (lcd_rs, lcd_e, lcd_d)
```

Complete los `// TODO` de la plantilla provista.

### Paso 4 — Conectar en `top.v`

Instancie los módulos y verifique que `state` y `cnt` del semáforo llegan correctamente a las entradas de `lcd_ctrl`.

### Paso 5 — Simular `lcd_ctrl.v`

Cree un testbench que:

- Fuerce la secuencia completa de inicialización.
- Cambie el valor de `state` tras la inicialización.
- Verifique en la forma de onda que `lcd_rs`, `lcd_e` y `lcd_d` generan los pulsos correctos.

Capture las formas de onda para el informe.

### Paso 6 — Síntesis e implementación

Ejecute el flujo completo en ISE. Revise el reporte de recursos y resuelva cualquier advertencia de timing antes de programar.

### Paso 7 — Demo en hardware

Programe la MAX 10 y demuestre simultáneamente al docente:

- Los LEDs cambian en secuencia VERDE → AMARILLO → ROJO → VERDE.
- La LCD muestra el estado correcto en la línea 1.
- El contador de segundos en la línea 2 decrece correctamente y se actualiza al cambiar de estado.


## Entregables

1. **Descripción de hardware** del proyecto en HDL (verilog).
2. **Documentación** del diseño en el archivo `README.md` del repositorio.
3. **Simulaciones** del sistema, con evidencias (capturas o formas de onda) incluidas en el `README.md`.
4. **Implementación física** de la descripción HDL sobre la tarjeta de desarrollo usando **Quartus IDE**, demostrando el funcionamiento con los periféricos requeridos durante la sesión de laboratorio.

*Técnicas Digitales — ECCI · Proyecto Final*