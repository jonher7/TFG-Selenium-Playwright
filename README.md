# TFG-Selenium-Playwright

Trabajo de Fin de Grado centrado en la **captura automatizada de tráfico web** (HAR + claves de sesión TLS), capturas de pantalla y control de parámetros de red/navegador (HTTP/1.1 vs HTTP/2, TLS 1.2 vs 1.3, user-agent), comparando dos aproximaciones de automatización de navegador: **Selenium** y **Playwright**, sobre tres navegadores basados en Chromium/Gecko: **Chrome**, **Brave** y **Firefox**.

El objetivo es visitar una URL de forma headless, registrar toda la actividad de red generada durante la carga en formato **HAR (HTTP Archive) v1.2**, volcar las claves de sesión TLS (`SSLKEYLOGFILE`) para poder descifrar ese tráfico posteriormente (p. ej. con Wireshark) y guardar una captura de pantalla de la página cargada.

## Estructura del repositorio

```
TFG_DEMO/
├── SELENIUM/                          # Implementación con Selenium (+ selenium-wire)
│   ├── chrome_selenium.py             # Captura HAR/SSL/screenshot con Chrome vía CDP
│   ├── brave_selenium.py              # Igual que el anterior pero con Brave
│   ├── firefox_selenium.py            # Captura con Firefox vía selenium-wire
│   ├── funciones.py                   # Utilidades básicas de creación/uso del driver
│   ├── configu.py                     # Construcción de opciones de Firefox desde fichero
│   ├── clean.py                       # Cierre controlado del driver
│   ├── conversor_har.py               # Conversión de requests de selenium-wire a HAR
│   ├── descarga_chrome.py             # Extracción de recursos (bodies) vía CDP (Chrome)
│   ├── descarga_brave.py              # Extracción de recursos (bodies) vía CDP (Brave)
│   ├── descarga_archivos.py           # Extracción de recursos vía selenium-wire (Firefox)
│   ├── prueba.py                      # Script de pruebas / ejemplo suelto
│   ├── opciones_chrome                # Flags de línea de comandos para Chrome
│   ├── opciones_brave                 # Flags de línea de comandos para Brave
│   ├── opciones_firefox               # Preferencias (about:config) para Firefox
│   ├── user_agent                     # Fichero de ejemplo con un user-agent a suplantar
│   ├── configuracion                  # Fichero de configuración auxiliar
│   ├── datos/                         # Salida: recursos descargados por visita
│   ├── har/                           # Salida: ficheros .har generados
│   ├── log/                           # Salida: claves SSL (SSLKEYLOGFILE) generadas
│   └── screenshot/                    # Salida: capturas de pantalla
│
└── PLAYWRIGHT/                        # Implementación equivalente con Playwright
    ├── prueba_playwright_google.chrome.py  # Captura con Chrome
    ├── prueba_playwright_brave.py           # Captura con Brave
    ├── prueba_playwright_firefox.py         # Captura con Firefox
    ├── opciones_chrome / opciones_brave     # Flags de línea de comandos (Chromium)
    ├── configuracion                        # Fichero de configuración auxiliar
    ├── datos/ · har/ · log/ · screenshot/   # Mismas carpetas de salida que en SELENIUM
```

Ambas carpetas replican el mismo experimento para poder comparar el comportamiento, las capacidades (p. ej. generación nativa de HAR en Playwright frente a la reconstrucción manual desde CDP/selenium-wire en Selenium) y el rendimiento de ambas herramientas.

## Qué hace cada script (resumen funcional)

1. Lanza el navegador indicado en **modo headless**, aplicando un conjunto de flags/preferencias pensadas para minimizar ruido de fondo (telemetría, caché, extensiones, crash reporter, etc.), definidas en los ficheros `opciones_*`.
2. Permite forzar la versión de **HTTP** (`h1`/`h2`) y de **TLS** (`1.2`/`1.3`) y, opcionalmente, suplantar el **user-agent** a partir de un fichero.
3. Exporta las claves de sesión TLS a través de la variable de entorno `SSLKEYLOGFILE`, de forma que el tráfico capturado se pueda descifrar más adelante.
4. Navega a la URL indicada y registra todas las peticiones/respuestas de red hasta que la página termina de cargar o se supera un umbral de inactividad (`--idle-threshold`).
5. Traduce lo capturado al formato estándar **HAR v1.2** y lo guarda en disco.
6. Guarda una **captura de pantalla** de la página (a página completa en los scripts que lo soportan).
7. Opcionalmente, vuelca a disco los **recursos individuales** (imágenes, JS, CSS, etc.) descomprimiendo `gzip`/`brotli` cuando procede.

### Diferencias clave entre implementaciones

- **Selenium + Chrome/Brave** (`chrome_selenium.py`, `brave_selenium.py`): usa el **Chrome DevTools Protocol (CDP)** directamente (`driver.execute_cdp_cmd`) para escuchar eventos de red (`Network.*`, `Page.*`) y reconstruir el HAR a mano.
- **Selenium + Firefox** (`firefox_selenium.py`): usa **selenium-wire**, ya que Firefox no expone CDP de la misma forma; el HAR se reconstruye a partir de `driver.requests`.
- **Playwright** (los tres scripts de `PLAYWRIGHT/`): usa la **generación nativa de HAR** de Playwright (`record_har_path`, `record_har_content`, `record_har_mode`), lo que simplifica notablemente el código frente a la aproximación de Selenium.

## Requisitos

- **Python 3.9+**
- Navegadores instalados localmente: Google Chrome, Brave y/o Firefox (las rutas a los binarios están **hardcodeadas para macOS**, ver [Limitaciones](#limitaciones-y-notas)).
- Dependencias de Python:

```bash
pip install selenium selenium-wire webdriver-manager psutil playwright brotli
python -m playwright install
```

> `selenium-wire` requiere una versión de `selenium` compatible (4.x) y puede dar problemas con versiones muy recientes de `blinker`/`urllib3`; si falla la instalación, fija versiones más antiguas de esas dos librerías.

> En Python 3.9 además hace falta `backports.zoneinfo` (usado en `chrome_selenium.py`, `clean.py` y `conversor_har.py`); en 3.9+ ya está en la librería estándar (`zoneinfo`), por lo que ese import puede eliminarse si se usa una versión más moderna.

## Uso

Todos los scripts se ejecutan desde dentro de su carpeta (`SELENIUM/` o `PLAYWRIGHT/`), porque leen ficheros de configuración (`opciones_chrome`, `opciones_firefox`, etc.) con ruta relativa.

### Selenium — ejemplo con Chrome

```bash
cd SELENIUM
python chrome_selenium.py \
  --url "https://www.example.com" \
  --http h2 \
  --tls 1.3 \
  --sslkeyfile log/claves.log \
  --har har/captura.har \
  --screenshot screenshot/captura.png \
  --idle-threshold 5000 \
  --timeout 30 \
  --ancho 1366 --altura 768 \
  --user_agent user_agent \
  --download True \
  --archivos_har datos
```

### Playwright — ejemplo con Firefox

```bash
cd PLAYWRIGHT
python prueba_playwright_firefox.py \
  --url "https://www.example.com" \
  --http h1 \
  --tls 1.2 \
  --sslkeyfile log/claves.log \
  --har har/captura.har \
  --screenshot screenshot/captura.png \
  --har_content embed \
  --ancho 1366 --altura 768
```

### Argumentos comunes

| Argumento | Obligatorio | Descripción |
|---|---|---|
| `--url` | Sí | URL a visitar |
| `--http` | No (`h2` por defecto) | Fuerza `h1` (HTTP/1.1) o `h2` (HTTP/2) |
| `--tls` | No | Fuerza versión TLS mínima/máxima: `1.2` o `1.3` |
| `--sslkeyfile` | Sí | Ruta donde volcar las claves de sesión TLS |
| `--har` | Sí | Ruta del fichero `.har` de salida |
| `--screenshot` | Sí | Ruta de la captura de pantalla |
| `--idle-threshold` | No (5000 ms) | Inactividad de red (ms) para dar por terminada la captura |
| `--timeout` | Solo Selenium (No, 30s) | Tiempo máximo de espera a que cargue la página |
| `--ancho` / `--altura` | No | Tamaño de la ventana/viewport |
| `--user_agent` | No | Ruta a un fichero con el user-agent a usar |
| `--archivos_har` | No | Carpeta donde volcar los recursos individuales descargados |
| `--download` | Solo `chrome_selenium.py`/`brave_selenium.py` | Si se activa, extrae también el body de cada recurso vía CDP |
| `--har_content` | Solo Playwright | `embed`, `attach` u `omit`: cómo incluir el contenido de los recursos en el HAR |

El nombre final del `.har` en los scripts de Selenium con CDP incluye un sufijo con el estado de la captura: `_loaded`, `_timeout_o_errores`, `_errores` o `_unreachable`.

## Limitaciones y notas

- Las rutas a los binarios de **Google Chrome** y **Brave** están fijadas en el código para **macOS** (`/Applications/...`). Para usarlo en Windows/Linux hay que editar esas rutas en los scripts correspondientes.
- `firefox_selenium.py` importa módulos (`configu`, `clean`, `conversor_har`, `descarga_archivos`) que también deben ejecutarse desde dentro de `SELENIUM/` para que las importaciones relativas funcionen.
- El proyecto contiene varios `TODO` dejados por el autor en el código (estimación de tamaños de cabeceras, timings exactos del HAR, etc.), señal de que es un trabajo en curso / demo del TFG y no una herramienta de producción.
- `prueba.py` es un script suelto de pruebas manuales sobre `funciones.py`, no forma parte del flujo principal de captura.
- Las carpetas `datos/`, `har/`, `log/` y `screenshot/` son los destinos de salida por defecto y se incluyen vacías en el repositorio.

## Autor

Proyecto de Trabajo de Fin de Grado — repositorio [`TFG-Selenium-Playwright`](https://github.com/jonher7/TFG-Selenium-Playwright).
# TFG-Selenium-Playwright
