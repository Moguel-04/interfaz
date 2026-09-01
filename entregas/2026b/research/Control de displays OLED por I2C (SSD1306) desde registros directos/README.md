# Control de displays OLED por I²C (SSD1306) desde registros directos

## Introducción

Las pantallas OLED basadas en el controlador SSD1306 son ampliamente utilizadas en sistemas embebidos debido a su bajo consumo, tamaño reducido y facilidad de comunicación. La interfaz I²C es una de las más comunes, ya que solo requiere dos líneas: SDA para datos y SCL para el reloj.

Controlar un SSD1306 desde registros directos permite entender el funcionamiento a bajo nivel, sin depender de bibliotecas como `display.text()` o `display.show()`. En este enfoque, el microcontrolador configura directamente los registros internos del periférico I²C y envía los bytes necesarios para inicializar el display, posicionar la memoria y escribir los píxeles.

En el caso del RP2040, este método permite estudiar el periférico I²C, sus registros de control y la estructura del protocolo utilizado por el SSD1306. El RP2040 cuenta con dos controladores I²C y múltiples GPIO configurables para esta función.

## ¿Qué es un display OLED SSD1306?

**OLED** (*Organic Light-Emitting Diode*) se diferencia de las pantallas LCD en que no necesita retroiluminación, ya que cada píxel genera su propia luz.

El **SSD1306** es un controlador para displays OLED monocromáticos, común en módulos de **128×64** y **128×32** píxeles.

### Componentes principales:
- Memoria gráfica GDDRAM
- Circuitos de control de píxeles
- Generador de temporización
- Control de contraste y alimentación
- Interfaz I²C o SPI

### Capacidad de memoria:
Para una pantalla de 128×64:
128 × 64 = 8192 píxeles
8192 / 8 = 1024 bytes

Se requieren **1024 bytes** de GDDRAM para representar toda la pantalla.

## Comunicación I²C

I²C es un protocolo serial síncrono que usa dos líneas:

| Línea | Función |
|-------|---------|
| SDA   | Datos seriales |
| SCL   | Reloj serial |

### Estructura de una transmisión típica:
START
  
  │
  
  ├── Dirección del dispositivo + R/W
  
  │
  
  ├── ACK
  
  │
  
  ├── Byte de datos
  
  │
  
  ├── ACK
  
  │
  
  ├── Byte de datos
  
  │
  
  ├── ACK
  
  │
  
  └── STOP

- **START**: SDA pasa de HIGH a LOW mientras SCL está HIGH.
- **STOP**: SDA pasa de LOW a HIGH mientras SCL está HIGH.

### Dirección I²C del SSD1306

Una confusión común es la diferencia entre dirección de 7 y 8 bits:

| Dirección | Valor | Descripción |
|-----------|-------|-------------|
| 7 bits    | 0x3C  | Dirección base |
| 8 bits    | 0x78  | Escritura (0x3C << 1) |
| 7 bits    | 0x3D  | Alternativa |
| 8 bits    | 0x7A  | Escritura alternativa |

## Estructura de transmisión al SSD1306

El SSD1306 utiliza un **control byte** para diferenciar comandos de datos:
START
↓
Dirección + WRITE
↓
Control Byte
↓
Comando/Datos
↓
...
↓
STOP


### Control Byte

| Co | D/C# | Función |
|----|------|---------|
| 0  | 0    | Comandos |
| 1  | 0    | Comandos con nuevo control byte |
| 0  | 1    | Datos gráficos |
| 1  | 1    | No utilizado |

- **D/C# = 0**: Los bytes siguientes son comandos
- **D/C# = 1**: Los bytes siguientes son datos gráficos (GDDRAM)

## Comandos principales

| Comando | Hex | Función |
|---------|-----|---------|
| Display OFF | 0xAE | Apaga el display |
| Display ON | 0xAF | Enciende el display |
| Set Contrast | 0x81 | Configura contraste |
| Display RAM | 0xA4 | Usa contenido de RAM |
| Display All ON | 0xA5 | Enciende todos los píxeles |
| Normal Display | 0xA6 | Modo normal |
| Invert Display | 0xA7 | Invierte píxeles |
| Set Multiplex | 0xA8 | Configura multiplexado |
| Set Offset | 0xD3 | Configura desplazamiento |
| Set Clock | 0xD5 | Configura reloj |
| Set Pre-charge | 0xD9 | Configura precarga |
| Set COM Pins | 0xDA | Configura pines COM |
| Set VCOMH | 0xDB | Configura nivel VCOMH |
| Charge Pump | 0x8D | Configura bomba de carga |
| Memory Mode | 0x20 | Modo de direccionamiento |

## Inicialización del SSD1306

Secuencia típica para 128×64:
START
Dirección + WRITE
Control = 0x00

AE → Display OFF
D5 80 → Clock
A8 3F → Multiplex 1/64
D3 00 → Offset
40 → Start Line
8D 14 → Charge Pump
20 00 → Horizontal Addressing
A1 → Segment Remap
C8 → COM Scan
DA 12 → COM Pins
81 7F → Contraste
D9 F1 → Pre-charge
DB 40 → VCOMH
A4 → RAM
A6 → Normal
AF → Display ON

STOP

Los valores pueden variar según el display. Cada instrucción se convierte en bytes enviados por I²C.

## GDDRAM y organización

La GDDRAM se organiza en **8 páginas de 128 bytes**:
8 × 128 = 1024 bytes

Cada byte controla **8 píxeles verticales** (un bit por píxel).

## Escritura de datos gráficos

Se envía el control byte `0x40` seguido de los datos:
START
0x78 (dirección)
0x40 (control byte)
0xFF (datos)
0xFF
0x00
0x00
STOP

## Control desde registros del RP2040

El RP2040 tiene periféricos I²C configurables por registros:

| Registro | Función |
|----------|---------|
| IC_CON | Configuración del controlador |
| IC_TAR | Dirección del dispositivo |
| IC_SAR | Dirección como target |
| IC_DATA_CMD | Datos y comandos |
| IC_SS_SCL_HCNT | Tiempo HIGH (Standard) |
| IC_SS_SCL_LCNT | Tiempo LOW (Standard) |
| IC_INTR_STAT | Estado de interrupciones |
| IC_STATUS | Estado del periférico |
| IC_ENABLE | Habilita/deshabilita I²C |

### IC_CON
Configura el modo maestro y la velocidad.

### IC_TAR
Contiene la dirección de 7 bits:
```c |```
IC_TAR = 0x3C;

IC_DATA_CMD
Permite enviar bytes con opciones de STOP/RESTART.

Envío de comandos:
START → 0x3C → 0x00 → 0xAE → STOP
Envío de datos:
START → 0x3C → 0x40 → Datos → STOP

## Configuración de los GPIO:

Los GPIO utilizados deben configurarse para funcionar como I²C. Se utilizan dos líneas principales:

SDA: transmisión de datos.
SCL: señal de reloj.

Además, el RP2040 y el OLED deben compartir GND.

## Flujo del sistema:
RP2040
  ↓
Registros I²C
  ↓
SDA / SCL
  ↓
SSD1306
  ↓
GDDRAM
  ↓
Píxeles OLED

El RP2040 configura los registros I²C y envía comandos o datos al SSD1306, que los utiliza para controlar la pantalla.

## Ventajas del control mediante registros directos:

* Comprensión del hardware
Permite observar la relación entre:
CPU
 ↓
Registros
 ↓
Periférico I²C
 ↓
Bus
 ↓
SSD1306
en lugar de ocultarla mediante una biblioteca.
* Mayor control:
-El programador puede modificar directamente:
-velocidad I²C;
-dirección del dispositivo;
-configuración del periférico;
-FIFO;
-condiciones START/STOP;
-transmisión de bytes;
-estado del controlador.

* Aprendizaje de sistemas embebidos:
-Este método es especialmente útil para comprender:
-registros de periféricos;
-mapas de memoria;
-buses seriales;
-protocolos de comunicación;
-máquinas de estados;
-acceso a hardware.

## Desventajas:

El principal inconveniente es la complejidad.
Con una biblioteca convencional podría bastar con:
Inicializar OLED
Mostrar texto
Actualizar pantalla
Mientras que mediante registros directos es necesario comprender:
GPIO
 ↓
I²C
 ↓
Dirección
 ↓
Control byte
 ↓
Comandos
 ↓
GDDRAM
 ↓
Píxeles
Además, un error en la configuración de los registros puede provocar:
-ausencia de comunicación;
-NACK;
-pantalla completamente apagada;
-datos desplazados;
-imagen incorrecta;
-bloqueo del bus.

Por esta razón, el acceso directo requiere consultar constantemente el datasheet del RP2040 y la documentación del SSD1306.

## Conclusión:
Controlar un SSD1306 desde registros directos permite entender a fondo cómo funciona un sistema embebido.

El proceso es:

-Configurar GPIO como I²C

-Configurar el periférico por registros

-Usar IC_TAR para la dirección

-Usar IC_DATA_CMD para enviar bytes

-Usar el control byte (0x00 para comandos, 0x40 para datos)

-Los datos se almacenan en GDDRAM

-El display actualiza los píxeles

Este enfoque no solo permite controlar el display, sino que es un excelente ejercicio para aprender sobre I²C, registros de hardware, 
memoria, comunicación serial y programación embebida en general.
