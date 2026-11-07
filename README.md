# JVM Fucker

Herramienta de apoyo para revisar Minecraft en un screenshare.

JVM Fucker examina el proceso `javaw.exe` del juego y busca argumentos de Java que pueden usarse para cargar componentes externos sin dejarlos en carpetas habituales como `mods`. Además reúne en un solo menú otras dos herramientas de análisis: **JVM-Inspector** y **Journal Trace**.

Sirve para responder preguntas como «¿este Minecraft se ha iniciado con un `-javaagent` o con una carpeta de mods distinta?». Algunos analizadores de mods (por ejemplo RedLotus Mod Analyzer o MeowModAnalyzer) solo miran los archivos de la carpeta de mods, y los argumentos personalizados de Java pueden quedar fuera de su alcance.

> **Importante:** un argumento desconocido **no confirma por sí solo el uso de cheats**. Los argumentos personalizados también tienen usos legítimos. Comprueba qué hace realmente, a qué archivo apunta y si existen otros indicadores que corroboren la sospecha.

## Requisitos

- Windows 10 u 11.
- Python 3.10 o superior.
- PyQt6: `pip install PyQt6`
- PowerShell (viene con Windows). Se usa para leer los procesos.
- Conexión a internet para las opciones 2 y 3.
- Permisos de administrador: recomendables para el escaneo y necesarios para las opciones 2 y 3 (la propia herramienta los pide con el aviso de UAC).

## Primer uso

1. Instala la dependencia: `pip install PyQt6`
2. Ejecuta `python jvm_fucker.py`, mejor desde una consola abierta como administrador.
3. Verás una intro con el texto **JVM-FUCKER** y después se carga el menú a pantalla completa. Un clic o una tecla salta la intro.
4. Con Minecraft abierto, elige **Escaneo de Java**.

El indicador de la parte superior del menú muestra si la herramienta se está ejecutando con permisos de administrador (`● ADMIN`) o sin ellos (`○ SIN ADMIN`).

## Controles

| Control | Qué hace |
|---|---|
| Clic o cualquier tecla durante la intro | Salta la intro |
| `—` (arriba a la derecha) | Minimiza la ventana |
| `✕` (arriba a la derecha) | Cierra el programa |
| `Esc` en la pantalla de escaneo | Vuelve al menú |
| Opción 4 · Exit | Cierra el programa |

La ventana es de pantalla completa, sin barra de título. Si necesitas cambiar de ventana durante la revisión, minimiza con `—`. Las animaciones se pausan mientras está minimizada.

## El menú

| Opción | Qué hace |
|---|---|
| **1 · Escaneo de Java** | Analiza el proceso `javaw.exe` y muestra los argumentos sospechosos que encuentre |
| **2 · JVM-Inspector** | Abre un CMD como administrador y ejecuta el script JVM-Inspector |
| **3 · Journal Trace** | Descarga la última versión de Journal Trace y la abre como administrador |
| **4 · Exit** | Cierra el programa |

Cada botón tiene su propio color: azul, morado, rojo y ámbar.

## Opción 1: Escaneo de Java

Pantalla dividida en dos partes:

- **Izquierda:** el estado del escaneo, el **PID** del proceso, su **hora de inicio** y un **contador de tiempo** que corre mientras se escanea y se detiene al terminar. Abajo están los botones **Reescanear** y **Volver**.
- **Derecha:** los resultados.

### Qué hace el escaneo

1. Consulta a Windows los procesos `javaw.exe` en ejecución (PowerShell y `Get-CimInstance Win32_Process`).
2. Muestra el PID y la hora de inicio de cada uno, con el tiempo que lleva abierto.
3. Lee su línea de comandos y la divide igual que lo hace Windows.
4. Busca los argumentos de la lista y analiza los archivos o carpetas a los que apuntan.
5. Muestra el resultado junto con la línea de comandos completa, para que puedas copiarla y revisarla.

Si hay varios `javaw.exe` abiertos, aparece un bloque de resultados por cada uno.

### Argumentos que detecta

| Argumento | Para qué puede usarse | Qué muestra la herramienta |
|---|---|---|
| `-javaagent` | Cargar un agente Java que puede instrumentar o modificar el comportamiento de la aplicación | Ruta del `.jar` y su estado |
| `-Dfabric.addMods` | Indicar mods adicionales de Fabric fuera de la carpeta habitual | Cada ruta (varias se separan con `;`) y su estado |
| `-Dfabric.modsFolder` | Cambiar la ubicación de la carpeta de mods | La carpeta, cuántos `.jar` contiene y sus nombres |
| `--gameDir` | Indicar un directorio de trabajo del juego distinto del habitual | La carpeta y su estado |

La búsqueda no distingue mayúsculas de minúsculas. Reconoce las formas `-argumento:valor`, `-argumento=valor` y, en el caso de `--gameDir`, también `--gameDir ruta`.

La sintaxis y el comportamiento pueden variar según el launcher, el loader y su versión. No todos estos argumentos funcionan de forma universal.

### Qué se comprueba de cada ruta

| Resultado | Significado |
|---|---|
| **archivo encontrado** | Existe. Se muestran el tamaño y el **SHA-256** (se omite en archivos de más de 300 MB) |
| **carpeta encontrada** | Existe. Se muestran cuántos `.jar` contiene y los nombres de los primeros 25 |
| **no existe o no es accesible** | No se encontró en este equipo (borrado, movido, ruta relativa o sin permisos) |
| **error al acceder** | Windows devolvió un error al leerla |

El SHA-256 sirve para comparar el archivo con una base de datos de hashes o con un análisis en línea. Si se trata de un `.jar`, comprueba también su contenido y su procedencia antes de sacar conclusiones.

### Ejemplo

```
javaw.exe  PID 4242 · inicio 2026-11-07 09:58:11 (hace 1 h 08 min)

2 argumento(s) detectado(s)

[!] -javaagent
Carga un agente Java que puede instrumentar o modificar el comportamiento de la aplicación.
Argumento: -javaagent:C:\Users\Jugador\Desktop\agent.jar=opcion
Ruta: C:\Users\Jugador\Desktop\agent.jar
archivo encontrado · 84.3 KB
SHA-256: 3f1c…e9a2

[!] --gameDir
Indica el directorio de trabajo del juego; Minecraft puede usar una carpeta distinta de la habitual.
Argumento: --gameDir D:\otro\.minecraft
Ruta: D:\otro\.minecraft
carpeta encontrada · 3 .jar
```

Si no aparece ninguno de los argumentos de la lista, el resultado lo indica con un mensaje verde. Eso **no garantiza** que el juego esté limpio: solo significa que no se han encontrado esos argumentos concretos.

## Opción 2: JVM-Inspector

Abre una ventana de CMD **como administrador** (Windows pide confirmación con el aviso de UAC) y ejecuta:

```
powershell -command "irm 'https://raw.githubusercontent.com/MeowTonynoh/Tonynoh-PS1/refs/heads/main/JVM-Inspector.ps1' | iex"
```

Para evitar problemas de comillas, la herramienta guarda ese comando en un archivo `.bat` temporal (dentro de una carpeta `JVMFucker_…` en `%TEMP%`) y lo ejecuta. La ventana del CMD queda abierta cuando termina el script para que puedas leer el resultado.

JVM-Inspector es un script de otra persona (Tonynoh) y puede revisar más cosas que JVM Fucker. Este programa solo lo lanza.

## Opción 3: Journal Trace

1. Descarga `JournalTrace.exe` de la **última versión** publicada en `github.com/spokwn/Journaltrace/releases` y la guarda en una carpeta temporal. Mientras descarga, el menú muestra el porcentaje.
2. La abre **como administrador** (aviso de UAC).

Si la descarga directa falla, la herramienta intenta localizar el `.exe` a través de la API de GitHub. Si tampoco funciona, abre la página de releases en el navegador para que la descargues a mano.

Journal Trace es un programa de otra persona (spokwn). JVM Fucker solo lo descarga y lo abre, y no lo incluye.

## Seguridad

- La opción 2 ejecuta **lo que haya en ese momento** en el repositorio indicado, con permisos de administrador. Úsala solo si confías en él.
- La opción 3 descarga un `.exe` de internet y lo ejecuta como administrador. Windows SmartScreen o tu antivirus pueden avisar, como con cualquier programa descargado de internet que no esté firmado. Descarga siempre de la página oficial y comprueba el archivo si lo necesitas.
- JVM Fucker no modifica el juego, no cierra procesos ni escribe nada en el sistema, salvo los archivos temporales de las opciones 2 y 3. El escaneo solo **lee** la información de los procesos y de los archivos.

## Limitaciones

- Solo funciona en **Windows**.
- Por defecto escanea únicamente `javaw.exe`. Si tu launcher usa `java.exe`, añádelo en `PROCESS_NAMES` (ver «Personalización»).
- Sin permisos de administrador, Windows puede impedir leer la línea de comandos de algunos procesos. Entonces el resultado lo indica.
- La lista de argumentos es fija y no pretende ser exhaustiva: no detecta otras formas de cargar código que no estén en la lista.
- No se expanden los archivos de argumentos (`@archivo`) que algunos launchers pasan a Java.
- Las rutas relativas no se pueden resolver, porque dependen de la carpeta de trabajo del proceso. Aparecen como «no existe o no es accesible».
- Si el archivo referenciado ya no existe en el disco, solo se puede mostrar la ruta.
- Se analiza el estado del proceso **en el momento del escaneo**. No registra lo que ocurrió antes de abrir la herramienta.

## Personalización

Todo está en el archivo `jvm_fucker.py`. Las opciones principales están al principio, en la sección de configuración:

| Constante | Qué controla |
|---|---|
| `PROCESS_NAMES` | Procesos que se escanean (por defecto `("javaw.exe",)`) |
| `DETECTIONS` | Lista de argumentos que se detectan, con su descripción y el tipo de ruta |
| `JVM_INSPECTOR_URL` | Dirección del script de JVM-Inspector |
| `JOURNAL_DIRECT_URL`, `JOURNAL_API_URL`, `JOURNAL_RELEASES_PAGE` | De dónde se descarga Journal Trace |
| `HASH_LIMIT_BYTES` | Tamaño máximo de archivo para calcular el SHA-256 (300 MB) |
| `MAX_JARS_LISTED` | Cuántos `.jar` de una carpeta se listan (25) |
| `COL_SCAN`, `COL_INSPECTOR`, `COL_JOURNAL`, `COL_EXIT`, `COL_BACK` | Colores de los botones |
| `PARTICLE_MAX`, `PARTICLE_COLORS` | Número máximo y colores de las partículas de fondo |

La duración de la intro se ajusta con `T_FADE` y `T_UI`, dentro de la clase `JvmFucker`.

## Solución de problemas

| Problema | Solución |
|---|---|
| «No hay ningún proceso javaw.exe en ejecución» | Abre Minecraft y pulsa **Reescanear**. Si tu launcher usa `java.exe`, añádelo a `PROCESS_NAMES` |
| «No se pudo leer la línea de comandos» | Ejecuta la herramienta como administrador y vuelve a escanear |
| El escaneo da un error de PowerShell | Comprueba que `powershell.exe` funciona en el equipo y que ninguna directiva lo bloquea |
| «No se pudo abrir el CMD como administrador (código 5)» | Se canceló o se denegó el aviso de UAC. Vuelve a pulsar el botón y acepta |
| Journal Trace no se descarga | Se abre la página de releases en el navegador. Descárgalo desde ahí |
| Windows SmartScreen o el antivirus bloquea un programa | Comprueba que lo descargas de la página oficial y, si confías en él, usa **Más información → Ejecutar de todas formas** |
| No consigo cerrar la ventana | Usa `✕`, la opción **4 · Exit** o `Alt + F4` |
| La interfaz va lenta | Las partículas consumen algo de CPU. Reduce `PARTICLE_MAX` o minimiza la ventana cuando no la uses |

## Créditos

- **JVM-Inspector:** script de Tonynoh, en el repositorio `MeowTonynoh/Tonynoh-PS1`.
- **Journal Trace:** programa de spokwn, en `spokwn/Journaltrace`.

JVM Fucker es independiente de ambos y no está afiliado con ellos.
