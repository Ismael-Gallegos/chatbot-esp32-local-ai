# Mapa de pines - ESP32 Chatbot Local AI

Este documento es la referencia maestra de qué GPIO del ESP32 se usa para cada componente. Se actualiza conforme se agregan nuevas piezas de hardware.

## Advertencia importante sobre la serigrafía

En el clon de ESP32-WROOM-32 usado en este proyecto, la serigrafía (texto impreso en la placa) de los pines es confiable, PERO no asumas que el LED integrado que ves encendido fijo es el LED programable — en esta placa hay un LED rojo de power (no conectado a ningún GPIO) y un LED azul programable en GPIO2/D2. Verifica siempre con `Serial.println()` en el Monitor Serial (115200 baudios) antes de confiar en observación visual únicamente.

## Componente: TFT ILI9341 2.4" (240x320, SPI, v1.3)

Módulo con pantalla ILI9341 + touch resistivo XPT2046 + lector MicroSD, todos compartiendo el mismo bus SPI físico.

|Pin del módulo TFT|Pin del ESP32|Notas|
|-|-|-|
|VCC|3V3|Lógica de 3.3V, no usar VIN (5V)|
|GND|GND||
|CS|D15|Chip Select de la pantalla|
|RESET|D4||
|DC|D2|Data/Command. Ver nota sobre GPIO2 abajo|
|SDI (MOSI)|D23|Bus SPI - hardware VSPI|
|SCK|D18|Bus SPI - hardware VSPI|
|LED (backlight)|3V3|Directo, sin PWM por ahora|
|SDO (MISO)|D19|Bus SPI - hardware VSPI|
|T\_CS|D33|Chip Select del touch (no probado aún)|
|T\_IRQ|D27|Interrupción del touch (no probado aún)|
|SD\_CS|D32|Chip Select de la MicroSD (no probado aún)|

Los pines T\_CLK, T\_DIN, T\_DO, SD\_SCK, SD\_MOSI, SD\_MISO NO se conectan por separado: comparten físicamente el mismo bus SPI (D18/D23/D19) que la pantalla, solo se distinguen por su línea CS individual.

### Nota sobre GPIO2/D2

GPIO2 se usó previamente para una prueba con LED externo (ver historial del proyecto). Se reutilizó para DC del TFT una vez confirmado que el ESP32, el IDE, y el flujo de subida funcionaban correctamente. Si se necesita repetir una prueba de LED simple en el futuro, usar otro GPIO libre.

### Configuración de librería (TFT\_eSPI de Bodmer)

Archivo editado: `User\_Setup.h` (dentro de la carpeta de la librería instalada)

```cpp
#define ILI9341\_DRIVER

#define TFT\_MOSI 23
#define TFT\_MISO 19
#define TFT\_SCLK 18
#define TFT\_CS   15
#define TFT\_DC    2
#define TFT\_RST   4

#define TOUCH\_CS 33

#define LOAD\_GLCD
#define LOAD\_FONT2
#define LOAD\_FONT4
#define LOAD\_FONT6
#define LOAD\_FONT7
#define LOAD\_FONT8
#define LOAD\_GFXFF
#define SMOOTH\_FONT

#define SPI\_FREQUENCY  40000000
#define SPI\_TOUCH\_FREQUENCY  2500000
```

### CRÍTICO: rotación de pantalla

Este módulo es nativamente vertical (240 ancho x 320 alto). La mayoría de ejemplos de TFT\_eSPI vienen pensados para 320x240 horizontal, lo cual causa síntomas confusos que parecen problema de cableado/ruido eléctrico (imagen "rayada", restos de dibujos anteriores pegados en pantalla, elementos que desaparecen en un borde) pero en realidad son de orientación de software.

**Solución confirmada:** usar siempre `tft.setRotation(0)` en el código para este módulo específico. No usar 1, 2, o 3.

```cpp
tft.begin();
tft.setRotation(0);  // Orientación correcta para este módulo 240x320 vertical
```

### Cómo se diagnosticó (para referencia futura)

1. Primer síntoma: imagen con "ruido" tipo estática de TV analógica en parte de la pantalla, números con líneas/rayas.
2. Hipótesis inicial (incorrecta): ruido eléctrico por velocidad SPI alta en cableado de protoboard. Se probó bajar de 40MHz a 20MHz y 10MHz sin ningún cambio — esto descartó la hipótesis de ruido eléctrico.
3. Hipótesis correcta: al probar un ejemplo de ping-pong, la pelota desaparecía al llegar a un borde específico y solo un \~75% de la pantalla se actualizaba correctamente — patrón clásico de desajuste de orientación/coordenadas, no de señal eléctrica.
4. Se probaron los 4 valores de `setRotation()` (0, 1, 2, 3). El valor 0 resolvió el problema por completo; el valor 2 mostraba la imagen recortada a la mitad.

## Componentes pendientes de mapear

* INMP441 (micrófono I2S)
* PAM8403 + altavoz (salida de audio)
* Touch del TFT (T\_CS, T\_IRQ ya reservados, falta probar funcionalmente)
* MicroSD del TFT (SD\_CS ya reservado, falta probar funcionalmente)

