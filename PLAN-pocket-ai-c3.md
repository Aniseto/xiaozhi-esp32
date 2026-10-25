# Plan: Firmware "Pocket AI Assistant" para Seeed XIAO ESP32-C3

> 📁 **Dónde vive este documento:** dentro del fork **`Aniseto/xiaozhi-esp32`**, rama `main`,
> en la raíz. Es un documento de trabajo personal — **nunca se sube al upstream
> `78/xiaozhi-esp32`**. Copia de trabajo también en `C:\Users\jordi\.opencode\plan\pocket-ai-c3-plan.md`.
>
> **Estado:** ✅ Fase 0 completada (entorno validado + 1ª compilación OK).
> ✅ Flash confirmado: **4 MB** · ✅ Pinout verificado visualmente con la datasheet oficial.
> **Siguiente paso:** **Fichero 1** (`main/boards/pocket-ai-c3/config.h`).
>
> **Regla del proyecto (permanente):** el asistente **NO modifica el código**. Explica QUÉ cambiar
> y POR QUÉ; **el usuario escribe y edita**, el asistente revisa antes de cada compilación.
>
> **Repositorio objetivo:** `C:\Users\jordi\xiaozhi-esp32` (commit `03a7c65` — *feat(boards): add StackChan*)
> **Código de referencia:** `C:\Users\jordi\Downloads\pocket-esp-ai-project-complete\POCKET-ESP-AI-PROJECT`
> **Fuente del proyecto original:** https://www.huyvector.org/robots-kinetic/pocket-ai-assistant
> **Datasheet de la placa:** `C:\Users\jordi\Downloads\esp32c3seed.pdf` (8 páginas, SKU 113991054)
> **ID de sesión:** `ses_e8bca0f01ffe6cOuQFOfGhZwse` · título *"Plan firmware Pocket AI C3 (Seeed XIAO ESP32-C3)"*

---

## 0. Contexto y objetivo

- Hardware: **Seeed Studio XIAO ESP32-C3**, OLED SSD1306 128x64 (I2C), micro INMP441 (I2S),
  amplificador MAX98357A (I2S), un botón externo.
- Objetivo a largo plazo: base para un **robot con IA** (de momento el robot no se mueve).
- Estrategia: partir del proyecto XiaoZhi original y **aprender a adaptarlo paso a paso**,
  usando el firmware ya adaptado de Huy Vector como "respuesta" / referencia, NO copiándolo
  ciegamente (su código es de una versión anterior del repo y ya no compila tal cual).

### Entorno instalado (Fase 0 ✅)

| Componente | Ruta / valor |
|---|---|
| ESP-IDF | `release/v6.1` en `C:\Users\jordi\esp\esp-idf` |
| Toolchains / Python env | `C:\Users\jordi\.espressif` · env `idf6.1_py3.14_env` |
| git | 2.51.1 |
| Compilación de prueba | `xmini/c3` → **`Project build complete`** ✅ (2114 targets) |
| `xiaozhi.bin` | `0x1f8540` = 2 061 632 B ≈ **2 013 KB** |
| Partición 4 MB `partitions/v2/4m.csv` → `factory` | `0x2F0000` = **3 008 KB** → **cabe con ~995 KB libres** ✅ |
| Binario de referencia (Huy Vector) | 3 541 630 B (merged, layout 4 MB) → assets ≈ 424 KB |
| **Flash de la placa** | **4 MB** (confirmado: datasheet *"400KB SRAM & 4MB Flash"*) |
| **SRAM** | **400 KB** (sin PSRAM → sin AFE → WakeNet9s) |
| Herramienta PDF | **PyMuPDF 2.0** + **pypdf 6.21.2** en `C:\Python314\python.exe` |
| Imágenes renderizadas | `C:\Users\jordi\AppData\Local\Temp\opencode\xiao_c3_p4.png`, `xiao_c3_p6.png` |

#### Comandos de arranque (una sesión nueva)

```powershell
call C:\Users\jordi\esp\esp-idf\export.bat
set PYTHONUTF8=1                       # evita el aviso de Unicode que salió en el build
cd /d C:\Users\jordi\xiaozhi-esp32
idf.py --version                       # debe imprimir v6.1
```

#### Herramienta PDF (leer datasheets, esquemas…)

```powershell
# Texto completo de un PDF
C:\Python314\python.exe -c "import pymupdf,sys;sys.stdout.reconfigure(encoding='utf-8');d=pymupdf.open('f.pdf');print(chr(10).join(p.get_text() for p in d))"
# Renderizar una página a PNG (el asistente puede VERLA)
C:\Python314\python.exe -c "import pymupdf;pymupdf.open('f.pdf')[0].get_pixmap(dpi=170).save('out.png')"
```

> Nota: el asistente **no puede leer PDFs directamente** (su modelo no admite PDF) → por eso
> instalamos PyMuPDF: extrae el texto y renderiza páginas a imagen.

---

## 1. Qué hizo exactamente Huy Vector (análisis)

Añadió **un solo board nuevo** (`pocket-wall-e-c3`, 5 ficheros) + **2 puntos de enganche**.
No reescribió XiaoZhi.

| Fichero | Función |
|---|---|
| `config.h` (24 líneas) | Pines: I2S mic/amp, I2C OLED, botón |
| `config.json` | target `esp32c3`, flash 4MB, partición `4m.csv`, wake word, idioma, consola USB |
| `pocket_wall_e_c3.cc` (175 líneas) | `class PocketWallEC3Board : public WifiBoard` + `DECLARE_BOARD(...)` |
| `pocket_oled_display.cc/h` | Cara de ojos animada en LVGL (parpadeo, miradas, estados) |
| `README.md` | Tabla de cableado |

Enganches en el resto del proyecto:

```
main/Kconfig.projbuild  →  config BOARD_TYPE_POCKET_WALL_E_C3 ... depends on IDF_TARGET_ESP32C3
main/CMakeLists.txt     →  elseif(CONFIG_BOARD_TYPE_POCKET_WALL_E_C3) / set(BOARD_DIR ...)
```

---

## 2. ⚠️ Las 6 diferencias: la referencia NO se puede copiar tal cual

Verificadas fichero a fichero contra el repo actual.

| # | En la referencia | En tu repo (actual) | Por qué / evidencia |
|---|---|---|---|
| 1 | `set(BOARD_TYPE "pocket-wall-e-c3")` | `set(BOARD_DIR "pocket-ai-c3")` | `BOARD_TYPE` ya no se pone en CMake: se lee del `"type"` de `config.json` → `main/CMakeLists.txt:940-951` |
| 2 | `config.json` con `CONFIG_LANGUAGE_EN_US`, `CONFIG_USE_ESP_WAKE_WORD`, `CONFIG_SR_WN_WN9S_HIESP` | **Prohibido** en `config.json` | `docs/custom-board.md:140-148`; `build.py` los inyecta solo (visible en el log de build) |
| 3 | `#include "audio/codecs/no_audio_codec.h"` | `#include "codecs/no_audio_codec.h"` | Cambió la ruta de includes |
| 4 | `void SetEmotion(const char*) override` | **`bool SetEmotion(...)`** | `main/display/display.h:50` → **error de compilación** si se copia el `void` |
| 5 | `std::string GetWifiHostnamePrefix() const override` | **No existe** en el repo (0 coincidencias) | **error de compilación** → hay que quitarlo |
| 6 | (no lo pone) | `BUILTIN_TEXT_FONT` debe ser `font_noto_sans_basic_<n>_<bpp>` | `CMakeLists.txt:1207` → `FATAL_ERROR`. `font_puhui_*` (que sugiere la doc) **fallaría** |

Otros cambios de API detectados en `main/display/`:
- `OledDisplay` ahora hereda de `LvglDisplay`.
- Apareció `RemoveChatMessage(...)` y `SetTheme(...)`.
- `SetStatus`, `SetEmotion`, `SetChatMessage`, `ClearChatMessages`, `SetupUI` siguen existiendo
  en la clase base `Display`.

---

## 3. Mapa de pines para Seeed XIAO ESP32-C3

> ### ✅ VERIFICADO con la datasheet oficial (pág. 6 "Hardware Overview", renderizada y vista)
>
> Pinout confirmado — **coincide 1:1 con la asignación propuesta más abajo**:
>
> ```
>    IZQUIERDA                         DERECHA
> D0  A0  GPIO2   D10 MOSI GPIO10
> D1  A1  GPIO3   D9  MISO GPIO9   (BOOT, ya en placa)
> D2  A2  GPIO4   D8  SCK  GPIO8
> D3  A3  GPIO5   D7  RX   GPIO20
> D4  SDA GPIO6   D6  TX   GPIO21
> D5  SCL GPIO7
> (derecha arriba) 5V · GND · 3V3
> ```
>
> **Ojo, esto también lo dice la datasheet:**
> 1. **No hay LED de usuario programable** — solo `CHARGE LED` (conectado al cargador, no a GPIO)
>    → **NO definir `BUILTIN_LED_GPIO` ni sobrescribir `GetLed()`**. `Board::GetLed()` ya tiene
>    implementación por defecto.
> 2. **Salida 3.3 V limitada a 500 mA.** OLED ≈20 mA + INMP441 ≈1 mA + chip ≈35 mA ≈ **56 mA** ✅
>    → **PERO los servos del robot NUNCA a 3.3 V** (SG90 pide 200-650 mA, picos >1 A):
>    van a **5 V (VIN)** con fuente externa y **GND común**. Dejarlo escrito en el README.
> 3. **Antena U.FL obligatoria** (la placa no trae PCB antenna) → sin ella el WiFi será muy malo.
> 4. Botones en placa: `RESET` + `BOOT` (GPIO9) → no cablear ninguno.
> 5. Spec: 21×17,8 mm · 11 GPIO(PWM), 1 UART, 1 IIC, 1 SPI, 4 ADC · WiFi activo 75 mA ·
>    deep sleep 44 μA · carga 380/40 mA · −40~85 °C · 160 MHz RISC-V.

### Pines disponibles (11)
`GPIO2, 3, 4, 5, 6, 7, 8, 9, 10, 20, 21`
(No existen en la XIAO: `GPIO0`, `GPIO1` ← **¡el GPIO1 de la referencia no existe!**;
`GPIO12-17` = flash; `GPIO18-19` = USB)

| Clasificación | Pines |
|---|---|
| Strapping (cuidado en reset) | `GPIO2` (no selecciona modo de arranque), `GPIO8` (debe estar ALTO), `GPIO9` (= botón BOOT de la placa) |
| **Pin "verde"** (sin restricción) | `GPIO3` (D1), `GPIO10` (D10) |
| JTAG por defecto (sin uso) | `GPIO4, 5, 6, 7` |
| UART0 por defecto (se libera) | `GPIO20, 21` |

### Regla que decide la asignación

En el arranque solo importa **qué pina puede ser tirada hacia abajo por algo EXTERNO**:

| Señal | ¿Quién la conduce? | Riesgo |
|---|---|---|
| I2S BCLK, WS, DOUT | el ESP (salida) → al reset queda flotando; mic y amp son entradas high-Z | ninguno |
| I2C SDA, SCL | open-drain con pull-ups → siempre ALTO | ninguno |
| **INMP441 `SD`** → entrada ESP | **el micro conduce SIEMPRE** | ⚠️ si cae en pin strapping → puede impedir el arranque |
| **Botón a GND** | el usuario puede pulsarlo durante el reset | ⚠️ ídem |

→ Las 2 señales peligrosas van a los 2 pines verdes.

### Asignación propuesta

| Función | GPIO | Pin XIAO | Por qué |
|---|---|---|---|
| I2S **DIN** ← INMP441 `SD` | **GPIO3** | D1 | 🟢 verde + única línea conducida externamente |
| **Botón** → GND | **GPIO10** | D10 | 🟢 verde → pulsarlo en reset sin romper nada |
| I2S **BCLK** (compartido) | **GPIO2** | D0 | salida ESP; su strapping no selecciona modo de arranque |
| I2S **WS** (compartido) | **GPIO4** | D2 | sustituye al GPIO1 inexistente; solo JTAG |
| I2S **DOUT** → MAX98357A `DIN` | **GPIO5** | D3 | salida ESP; JTAG MTDI sin uso |
| I2C **SDA** → OLED | **GPIO6** | D4 | **I2C nativo de la XIAO** (default Seeed/Arduino) |
| I2C **SCL** → OLED | **GPIO7** | D5 | ídem |
| **libres** | GPIO8, GPIO20, GPIO21 | D8, D7, D6 | **3 pines para el robot** (servos/sensores) |

### Cableado complementario (física)

- `INMP441 L/R` → **GND** → canal izquierdo → coincide con `I2S_STD_SLOT_LEFT` de `NoAudioCodecDuplex` ✅
- `MAX98357A SD` (pin de formato) → **GND** = solo canal L.
  Si flota → (L+R)/2 y como R está en silencio → **-6 dB de volumen**.
- `MAX98357A` a **5V** (pin 5V de la XIAO); OLED y micro a **3.3V**; **GND común a todos**.
- Verificar flash antes de nada: `python -m esptool --chip esp32c3 flash_id`

---

## 4. La cadena de configuración (idea central)

```
config.json ──lee──> build.py ──parsea──> CMakeLists.txt (rama if/elseif) ──fija──> BOARD_DIR
     │                                                                                   │
     └── "type"/"name" ──────────────> -DBOARD_NAME ────────────────────>  boards/<dir>/ │
                                                                                        ▼
Kconfig.projbuild ──elige SÍMBOLO──> decide QUÉ ramas de CMake entran ──> config.h + .cc (DECLARE_BOARD)
```

**Errores concretos si falta un eslabón:**

| Falta | Error |
|---|---|
| Rama en `CMakeLists.txt` | `build.py`: *"Cannot find board config symbol for ..."* (función `_find_board_config_candidates()` lee `main/CMakeLists.txt` buscando `set(BOARD_DIR "...")`, `scripts/build.py:1108-1144`) |
| Entrada en `Kconfig.projbuild` | *"does not exist in Kconfig"* |
| `depends on IDF_TARGET_ESP32C3` | *"does not support target 'esp32c3'"* (`_symbol_supports_target`) |
| `DECLARE_BOARD` | *undefined reference a `create_board()`* |
| Dos `DECLARE_BOARD` | *duplicate symbol* |

### El mecanismo `DECLARE_BOARD` (`main/boards/common/board.h:62-90`)

```cpp
void* create_board();          // declarada aquí, sin definir

class Board {
public:
    static Board& GetInstance() {
        static Board* instance = static_cast<Board*>(create_board());  // la llama
        return *instance;
    }
    virtual AudioCodec* GetAudioCodec() = 0;   // OBLIGATORIO implementar
    virtual Display*    GetDisplay();          // opcional (tiene default)
    ...
};

#define DECLARE_BOARD(BOARD_CLASS_NAME) \
void* create_board() { return new BOARD_CLASS_NAME(); }
```

CMake solo añade **un** directorio de board (`BOARD_DIR`) → solo hay una definición de
`create_board()` → enlace limpio. Eso es la regla *"exactamente un `DECLARE_BOARD` por build"*.

**Métodos `= 0` (obligatorios):** `GetBoardType`, `GetAudioCodec`, `GetNetwork`, `StartNetwork`,
`GetNetworkStateIcon`, `SetPowerSaveLevel`, `GetBoardJson`, `GetDeviceStatusJson`.
**Opcionales:** `GetDisplay`, `GetBacklight`, `GetLed`, `GetCamera`, `GetBatteryLevel`.

---

## 4b. Verificaciones técnicas HECHAS (ya no hay que dudar de esto)

### (a) Nuestro `sdkconfig_append` GANA sobre los defaults ✅

Riesgo real: `sdkconfig.defaults` pone **16 MB** y `sdkconfig.defaults.esp32c3` pone **16m_c3.csv**.

Código real de ESP-IDF (`tools/cmake/kconfig.cmake:187-193`):

```cmake
foreach(sdkconfig_default ${sdkconfig_defaults})
    list(APPEND defaults_arg --defaults "${sdkconfig_default}")
    if(EXISTS "${sdkconfig_default}.${idf_target}")
        list(APPEND defaults_arg --defaults "${sdkconfig_default}.${idf_target}")  # autoañadido
    endif()
endforeach()
```

`build.py:1444-1450` pasa → `sdkconfig.defaults` ; `build/xiaozhi-build.sdkconfig.defaults`

**Orden efectivo (gana el último):**

```
1. sdkconfig.defaults                          (16MB, 16m.csv)
2. sdkconfig.defaults.esp32c3                  (16m_c3.csv)  ← autoañadido
3. build/xiaozhi-build.sdkconfig.defaults      (TU sdkconfig_append) ← GANA ✅
```

**Prueba empírica** (leído del `sdkconfig` generado por el build de prueba):

```
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions/v2/16m_c3.csv"   ← sí se aplica el .esp32c3
CONFIG_COMPILER_OPTIMIZATION_SIZE=y                                 ← ya activo
CONFIG_ESP_WIFI_STATIC_RX_BUFFER_NUM=3                              ← ya activo
CONFIG_FREERTOS_IDLE_TASK_STACKSIZE=768                             ← ya activo
```

### (b) Las 10 líneas de tuning de la referencia → **0 necesarias**

| Lo que la referencia añade | ¿Tu repo ya lo tiene? |
|---|---|
| `COMPILER_OPTIMIZATION_SIZE=y` | ✅ `sdkconfig.defaults` |
| `NEWLIB_NANO_FORMAT=y` | ✅ `sdkconfig.defaults` |
| `ESP_WIFI_ENTERPRISE_SUPPORT=n` | ✅ `sdkconfig.defaults` |
| `ESP_WIFI_STATIC_RX_BUFFER_NUM=4` | ✅ **3** (más agresivo) |
| `ESP_WIFI_DYNAMIC_RX_BUFFER_NUM=6` | ✅ 6 |
| `ESP_WIFI_RX_BA_WIN=4` | ✅ **3** (más agresivo) |
| `ESP_WIFI_ENABLE_WPA3_SAE=n` | ✅ `sdkconfig.defaults.esp32c3` |
| `LWIP_IPV6=n` | ✅ `sdkconfig.defaults.esp32c3` |
| `LWIP_TCPIP_RECVMBOX_SIZE=16` | ✅ ídem |
| `MBEDTLS_DYNAMIC_FREE_CONFIG_DATA=y` | ✅ ídem |
| `FREERTOS_IDLE_TASK_STACKSIZE=768` | ✅ ídem |

→ **El `config.json` solo necesita 3 líneas**, no 16 como el suyo.

### (c) 400 KB SRAM → 3 consecuencias

| # | Consecuencia |
|---|---|
| 1 | **Wake word = WakeNet9s "Hi ESP"** — sin PSRAM no hay AFE (`docs/custom-board.md:400`).
     `build.py` lo selecciona solo y **rechaza** modelos AFE antes de configurar. |
| 2 | **Sin AEC por hardware** → eco residual en salida de altavoz. → **NO** poner `CONFIG_USE_DEVICE_AEC` en Fase A. |
| 3 | Ya todo ajustado (tabla (b)) + 10 boards C3 ya probados en el repo (`xmini/c3`, `kevin/c3`, `magiclick/c3`, `lckfb/szpi-esp32c3`). |

### (d) ⚠️ Con 4 MB **NO habrá OTA** (actualización inalámbrica)

```
sdkconfig.defaults.esp32c3 → 16m_c3.csv:  ota_0 (0x20000) + ota_1  ← A/B, doble slot
partitions/v2/4m.csv       →  factory (0x10000)  ← SLOT ÚNICO
```

| Escenario | Tamaño máx. de app | Tu app |
|---|---|---|
| 4 MB doble slot OTA (−64 KB cabecera −1 MB assets) | `(0x400000 − 0x10000 − 0x100000)/2` = **1 474 KB** | 2 013 KB ❌ |
| Ídem con assets a 512 KB | **1 773 KB** | 2 013 KB ❌ |
| **`4m.csv` slot único** (`factory` 0x2F0000) | **3 008 KB** | 2 013 KB ✅ |

→ **Actualizar = flashear por USB** (igual que la referencia). Dejarlo anotado en el README:
*"No OTA: partición `factory` única por límite de 4 MB; actualización solo por USB."*

### (e) Cuentas reales de cabida (4 MB exactos)

```
APPLICACIÓN   0x10000 + 0x1F8540 = 0x208540   <  0x300000  ✅  sobran 991 KB
ASSETS        0x300000 + 0x060A88 = 0x360A88  <  0x400000  ✅  sobran 637 KB
```

---

## 5. Los 5 ficheros a crear/editar

### Cadena recomendada

| # | Fichero | Acción |
|---|---|---|
| 1 | `main/boards/pocket-ai-c3/config.h` | NUEVO |
| 2 | `main/boards/pocket-ai-c3/config.json` | NUEVO |
| 3 | `main/boards/pocket-ai-c3/pocket_ai_c3.cc` | NUEVO |
| 4 | `main/Kconfig.projbuild` | EDITAR (+3 líneas) |
| 5 | `main/CMakeLists.txt` | EDITAR (+3 líneas) |
| 6 | `main/boards/pocket-ai-c3/README.md` | NUEVO (regla del repo) |

**Nombre propuesto:** directorio `pocket-ai-c3`, `type: "pocket-ai-c3"`, variante `pocket-ai-c3`.
⚠️ `type`/`name` son **identidad estable tras publicar** (rompe OTA si se cambian después).
Solo se admiten `minúsculas`, `dígitos`, `.` y `-`.

---

### Fichero 1 · `main/boards/pocket-ai-c3/config.h`

Solo `#define`s de pines. Existe para aislar el hardware en un único sitio
(nunca hay que tocar otro board).

```c
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_

#include <driver/gpio.h>

/* --- Audio: I2S full-duplex con RELOJ COMPARTIDO --- */
/* El C3 tiene UN solo controlador I2S -> micro y ampliador comparten BCLK/WS,
   por eso IN y OUT deben ir a la misma frecuencia (24 kHz). */
#define AUDIO_INPUT_SAMPLE_RATE  24000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000

#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_2   /* INMP441 SCK  + MAX98357A BCLK */
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_4   /* INMP441 WS   + MAX98357A LRC  */
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_5   /* -> MAX98357A DIN              */
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_3   /* <- INMP441 SD                 */

/* --- OLED SSD1306 128x64 I2C (0x3C) --- */
#define DISPLAY_SDA_PIN GPIO_NUM_6
#define DISPLAY_SCL_PIN GPIO_NUM_7
#define DISPLAY_WIDTH   128
#define DISPLAY_HEIGHT  64
#define DISPLAY_MIRROR_X false
#define DISPLAY_MIRROR_Y false

/* --- Botón externo a GND (Button usa pull-up interno) --- */
#define ACTION_BUTTON_GPIO GPIO_NUM_10

#endif /* _BOARD_CONFIG_H_ */
```

**Por qué no hace falta `BUILTIN_LED_GPIO` / `BOOT_BUTTON_GPIO`:** verificado que
`#include "config.h"` solo ocurre dentro de `main/boards/**`. Ningún fichero del núcleo
lee tu `config.h` → defines exactamente lo que usa **tu** `.cc`.

**Por qué tampoco hay `BUILTIN_LED_GPIO`:** la datasheet (pág. 6) muestra solo un
`CHARGE LED` conectado al cargador, **no a ningún GPIO** → la XIAO ESP32-C3 **no tiene
LED de usuario programable**. No definas la macro ni sobrescribas `GetLed()`
(`Board::GetLed()` ya tiene implementación por defecto y compila sin tocarlo).

---

### Fichero 2 · `main/boards/pocket-ai-c3/config.json`

```json
{
    "type": "pocket-ai-c3",
    "target": "esp32c3",
    "builds": [
        {
            "name": "pocket-ai-c3",
            "sdkconfig_append": [
                "CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y",
                "CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/4m.csv\"",
                "CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y"
            ]
        }
    ]
}
```

| Línea | Por qué |
|---|---|
| `"type"` | Identidad OTA. CMake lo lee (`:949`) y lo reporta como `BOARD_TYPE` |
| `"target": "esp32c3"` | Hace `idf.py set-target esp32c3` |
| `FLASHSIZE_4MB` | La XIAO tiene 4 MB; default del repo = 16 MB |
| `partitions/v2/4m.csv` | `factory 0x10000+0x2F0000`, `assets 0x300000+0x100000`. App 2013 KB **cabe** |
| `ESP_CONSOLE_USB_SERIAL_JTAG` | Libera GPIO20/21 (UART0) y da monitor por el mismo cable USB |
| ❌ idioma / wake word | Prohibido por norma → `build.py --language` / `--wake-word` |
| ❌ `CONFIG_BOARD_TYPE_...=y` | `build.py` lo deriva solo de la rama de CMake |

Contenido de `partitions/v2/4m.csv`:

```
nvs,      data, nvs,     0x9000,    0x4000,
otadata,  data, ota,     0xd000,    0x2000,
phy_init, data, phy,     0xf000,    0x1000,
factory,  app,  factory, 0x10000,   0x2F0000,
assets,   data, spiffs,  0x300000,  0x100000,
```

---

### Fichero 3 · `main/boards/pocket-ai-c3/pocket_ai_c3.cc`

**FASE A — versión mínima** (estructura tomada de `bread-compact-wifi` y `xmini/c3`):

```cpp
#include "wifi_board.h"
#include "codecs/no_audio_codec.h"      // ← ruta ACTUAL (no "audio/codecs/...")
#include "display/oled_display.h"
#include "application.h"
#include "button.h"
#include "config.h"

#include <driver/i2c_master.h>
#include <esp_lcd_panel_ops.h>
#include <esp_lcd_panel_vendor.h>
#include <esp_log.h>

#define TAG "PocketAiC3"

class PocketAiC3Board : public WifiBoard {
private:
    i2c_master_bus_handle_t display_i2c_bus_ = nullptr;
    esp_lcd_panel_io_handle_t panel_io_ = nullptr;
    esp_lcd_panel_handle_t panel_ = nullptr;
    Display* display_ = nullptr;
    Button action_button_;

    void InitializeDisplayI2c() { /* copiar el de xmini/c3 con DISPLAY_SDA/SCL_PIN */ }
    void InitializeDisplay()    { /* SSD1306 0x3C, 400 kHz, asignar campo a campo */ }

    void InitializeButton() {
        action_button_.OnClick([this]() {
            auto& app = Application::GetInstance();
            if (app.GetDeviceState() == kDeviceStateStarting) { EnterWifiConfigMode(); return; }
            app.ToggleChatState();
        });
        action_button_.OnLongPress([this]() { EnterWifiConfigMode(); });
    }

public:
    PocketAiC3Board() : action_button_(ACTION_BUTTON_GPIO) {
        InitializeDisplayI2c();
        InitializeDisplay();
        InitializeButton();
    }

    AudioCodec* GetAudioCodec() override {
        static NoAudioCodecDuplex audio_codec(
            AUDIO_INPUT_SAMPLE_RATE, AUDIO_OUTPUT_SAMPLE_RATE,
            AUDIO_I2S_GPIO_BCLK, AUDIO_I2S_GPIO_WS,
            AUDIO_I2S_GPIO_DOUT, AUDIO_I2S_GPIO_DIN);
        return &audio_codec;
    }

    Display* GetDisplay() override { return display_; }
};

DECLARE_BOARD(PocketAiC3Board);
```

**Porqués importantes:**

- `io_config = {}` **y luego asignar campo a campo** (no designados `{.x = ...}`): en IDF 6
  el orden de campos cambió respecto a 5.5 → así es inmune (lo documenta `xmini/c3:97`).
- `display_i2c_bus_` como **miembro**, no local: si el bus muere al salir de la función,
  el OLED deja de responder.
- `static NoAudioCodecDuplex` dentro de `GetAudioCodec()`: se construye **una sola vez**.
  Si estuviera en el constructor se recrearía y `i2s_new_channel` abortaría.
- **NO** `GetWifiHostnamePrefix()` → no existe en este repo.
- `ESP_ERROR_CHECK` en el I2C: reinicio en bucle si el OLED está mal → *deseado* en bring-up.

**Sobre `NoAudioCodec` (jerarquía, `main/audio/`):**

```
AudioCodec  (audio_codec.h)
   protected: tx_handle_, rx_handle_, output_volume_, input_gain_, ...
   virtual int Read(...)  = 0
   virtual int Write(...) = 0
     └─ NoAudioCodec (codecs/no_audio_codec.*)   I2S "sin chip codec"
          ├─ NoAudioCodecDuplex   ← 1 controlador I2S, RX+TX compartidos ✅
          ├─ NoAudioCodecSimplex  ← 2 controladores (placa grande)
          └─ NoAudioCodecSimplexPdm
```

- `rx_handle_` es `protected` → por eso una clase interna puede usar
  `i2s_channel_read(rx_handle_, ...)` sin getter (herencia).
- **Diferencia de escala del micro** (el porqué del AGC de la referencia):
  - Base (`NoAudioCodec::Read`): `value = bit32_buffer[i] >> 12`
  - Referencia (`PocketAudioCodec::Read`): `input = raw[i] >> 14` + filtro paso alto + AGC
  - Filtro: `input - prev_in + (prev_out * 32604 >> 15)` → quita DC y ronroneo del recinto
    (**muy útil cuando haya servos/motores** en el robot).
  - AGC: 0.5x..6x, pico objetivo ≈ -7.4 dBFS; suave al subir, rápido al bajar
    (`gain = (gain + target*3)/4` al subir, `(gain*7 + target)/8` al bajar).
- **Por qué 24 kHz en ambos sentidos:** el C3 tiene **un** controlador I2S →
  micro y ampliador comparten `BCLK`/`WS` → no se puede 16 kHz a un lado y 24 al otro.
  Solución: 24 kHz en ambos y `AudioService` re-muestrea a 16 kHz para WakeNet/nube.

---

### Fichero 4 · `main/Kconfig.projbuild` (+3 líneas)

Dentro de `choice BOARD_TYPE`, **sin tocar las demás entradas ni el `default`**:

```kconfig
    config BOARD_TYPE_POCKET_AI_C3
        bool "Pocket AI Assistant (Seeed XIAO ESP32-C3)"
        depends on IDF_TARGET_ESP32C3
```

| Fragmento | Por qué |
|---|---|
| MAYÚSCULAS + `_` | Convención; `build.py:1248` genera `CONFIG_BOARD_TYPE_` + nombre en mayúsculas |
| `bool` | Dentro de un `choice` solo puede ser `bool` |
| `depends on IDF_TARGET_ESP32C3` | **Obligatorio** → `_symbol_supports_target()` lo valida |

---

### Fichero 5 · `main/CMakeLists.txt` (+3 líneas)

```cmake
elseif(CONFIG_BOARD_TYPE_POCKET_AI_C3)
    set(BOARD_DIR "pocket-ai-c3")
    set(BUILTIN_TEXT_FONT font_noto_sans_basic_14_1)
    set(BUILTIN_ICON_FONT font_material_symbols_14_1)
```

- `BOARD_DIR` **debe coincidir exactamente** con el directorio: es la llave con la que
  `build.py` encuentra esta rama.
- Fuentes `14_1` = estándar de los OLED 128x64 (`bread-compact-wifi`, `xmini/c3`).
  **Solo vale el patrón `font_noto_sans_basic_<n>_<bpp>`.**
- **No** `set(BOARD_NAME ...)`: `build.py` ya pasa `-DBOARD_NAME=pocket-ai-c3`
  (`scripts/build.py:1451`) y una `set()` en CMake lo sobrescribiría.

---

### Fichero 6 · `main/boards/pocket-ai-c3/README.md`

Obligatorio según `docs/custom-board.md` paso 6. Debe incluir:

- **Tabla de cableado** (la de la sección 3, con los GPIO definitivos).
- **Alimentación:**
  - OLED + INMP441 → 3V3 (**límite 500 mA** del regulador de la XIAO; consumo actual ≈56 mA)
  - MAX98357A → **5V (VIN)**
  - **Servos/robot → 5V con fuente EXTERNA, nunca desde 3V3**, GND común con la XIAO
  - Todos los módulos comparten **GND**
- **`INMP441 L/R` → GND** (canal izquierdo, casa con `I2S_STD_SLOT_LEFT`)
- **`MAX98357A SD` → GND** (solo canal L; si flota → −6 dB de volumen)
- **Antena U.FL** conectada antes de probar WiFi
- **Sin LED de usuario** (solo CHARGE LED)
- **⚠️ No OTA** — partición `factory` única por límite de 4 MB; actualización solo por USB
- Verificación previa: `python -m esptool --chip esp32c3 flash_id` (debe decir 4 MB)

---

## 6. Secuencia de validación

```powershell
# 1) Verificar que build.py "ve" el board (solo parseo, no compila)
python scripts/build.py --list-boards | findstr /i pocket

# 2) FASE A: OLED plano + NoAudioCodecDuplex plano
python scripts/build.py pocket-ai-c3

# 3) FASE B: display de ojos (PocketOledDisplay adaptado -> ojo al bool SetEmotion)
python scripts/build.py pocket-ai-c3

# 4) FASE C: filtro DC + AGC en el codec
python scripts/build.py pocket-ai-c3

# 5) Flashear
python -m esptool --chip esp32c3 flash_id
python -m esptool --chip esp32c3 -b 460800 write_flash 0x0 build\merged-binary.bin
```

| Fase | Objetivo | Señal de éxito |
|---|---|---|
| **A** | Arranque + WiFi + texto en OLED | log `PocketAiC3Board`, pantalla de estado, botón abre config WiFi |
| **B** | Ojos animados | `PocketOledDisplay` compila (¡`bool SetEmotion`!) |
| **C** | Voz nítida | wake word "Hi ESP" responde, sin ronroneo DC |

> La compilación **no** valida los pines. Si entra en bucle de reinicio al flashear →
> casi seguro OLED mal cableado (address/reset) o un pin strapping forzado en el arranque.

---

## 7. Roadmap completo a largo plazo

| # | Paso | Aprendizaje |
|---|---|---|
| 0 | ~~Instalar ESP-IDF + compilar board existente~~ | ✅ hecho |
| 1 | Entender `Application` / `Board` / `AudioCodec` / `Display` (solo lectura) | Arquitectura de capas |
| 2 | Crear carpeta del board con `config.json` + `config.h` | Cadena de configuración |
| 3 | **Asignar pines de la XIAO** + strapping/UART | Electricidad + porqué |
| 4 | Escribir la clase board (`: public WifiBoard` + `DECLARE_BOARD`) | Herencia e inicialización |
| 5 | Enganchar en `Kconfig.projbuild` + `CMakeLists.txt` | Los 2 eslabones del build |
| 6 | Compilar con `scripts/build.py` | Build de variantes |
| 7 | Audio: `NoAudioCodecDuplex`, 24 kHz → 16 kHz | Pipeline de audio |
| 8 | Display: carita de ojos LVGL | UI + estados |
| 9 | Wake word `Hi ESP`, idioma, nombre de dispositivo | Kconfig de usuario |
| 10 | Flashear y monitor serie | Bring-up |
| 11 | *(futuro)* Servos/motores del robot → MCP `Servo`/`GPIO` | Expansión |

---

## 8. Preguntas abiertas (bloquean el inicio)

1. ~~**¿Flash = 4 MB?**~~ ✅ **RESUELTA** — datasheet oficial: *"On-chip Memory: 400KB SRAM & 4MB Flash"*
   → usar `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions/v2/4m.csv"` y `FLASHSIZE_4MB`.
2. **¿Nombre del board?** Propuesto: `pocket-ai-c3` (directorio, `type`, variante).
   Es un valor **estable tras publicar** (rompe OTA si se cambia) → **decidirlo antes de compilar**.
   ⚠️ Recordar que con 4 MB no hay OTA, así que la "estabilidad" solo afecta a la identidad reportada.
3. **¿Empezar por la Fase A** (mínimo compilable y flasheable) y luego ojos + AGC?
   Recomendado: cada fase tiene un fallo localizable.

---

## 9. Cómo retomar esta sesión otro día

1. Abrir OpenCode en `C:\Users\jordi\xiaozhi-esp32`.
2. Recuperar la sesión → título **"Plan firmware Pocket AI C3 (Seeed XIAO ESP32-C3)"**
   (ID `ses_e8bca0f01ffe6cOuQFOfGhZwse`).
3. Leer este documento: `C:\Users\jordi\.opencode\plan\pocket-ai-c3-plan.md`
4. Comprobar que sigue instalado:
   ```powershell
   call C:\Users\jordi\esp\esp-idf\export.bat
   idf.py --version                     # v6.1
   C:\Python314\python.exe -c "import pymupdf; print(pymupdf.__doc__)"   # PyMuPDF 2.0
   ```
5. Decidir las **2 preguntas que quedan** de la sección 8 (nombre del board · empezar por Fase A).
6. Empezar por el **Fichero 1** (`config.h`), con el asistente revisando cada fichero
   **antes** de compilar.

### Resumen de dónde estamos

| Hecho | Pendiente |
|---|---|
| ✅ ESP-IDF v6.1 instalado y validado con una compilación real | ⬜ Fichero 1 `config.h` |
| ✅ Análisis del proyecto de referencia + sus 6 incompatibilidades | ⬜ Fichero 2 `config.json` |
| ✅ Datasheet leído (PyMuPDF): 4 MB / 400 KB / 11 GPIO | ⬜ Fichero 3 `pocket_ai_c3.cc` (Fase A) |
| ✅ Pinout verificado visualmente | ⬜ Fichero 4 `Kconfig.projbuild` |
| ✅ Precedencia del `sdkconfig` verificada | ⬜ Fichero 5 `CMakeLists.txt` |
| ✅ Límites de RAM, AFE y ausencia de OTA calculados | ⬜ Fichero 6 `README.md` |
| | ⬜ Compilar Fase A |

> **Recordatorio permanente:** el asistente NO edita el código del proyecto.
> Explica qué cambiar y por qué; el usuario lo escribe.
