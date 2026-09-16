# GRATMA Multipuerto

Script en Python para realizar **medidas I-V con uno o varios dispositivos GRATMA en paralelo** mediante distintos puertos serie.

Cada GRATMA funciona de forma independiente en su propio hilo. Los 8 sensores se miden siguiendo un orden aleatorio y los resultados se guardan automáticamente en una carpeta diferente para cada chip.

## Requisitos

Es necesario tener Python instalado junto con la librería:

```bash
pip install pyserial
```

El resto de módulos utilizados pertenecen a la librería estándar de Python.

## Parámetros de medida

Los principales parámetros se pueden modificar al comienzo del código:

```python
VD = 50
VGINIT = 0
VGEND = 1200
VGSWEEP = 15
FBWD = 1
NUM_SEQUENCES = 5
```

Donde:

- **VD**: tensión utilizada durante la medida, en mV.
- **VGINIT**: tensión inicial del barrido, en mV.
- **VGEND**: tensión final del barrido, en mV.
- **VGSWEEP**: paso de tensión del barrido, en mV.
- **FBWD**:
  - `0`: solamente barrido forward.
  - `1`: barrido forward y backward.
- **NUM_SEQUENCES**: número de secuencias completas sobre los 8 sensores.

También se pueden modificar los tiempos:

```python
STABILIZE_S = 600
BETWEEN_SENSORS_S = 10
```

Se realiza una estabilización inicial de **600 segundos** y una espera de **10 segundos entre sensores**.

## Carpeta de salida

La carpeta principal donde se guardan las medidas se define mediante:

```python
FOLDER_PATH = r"C:\ruta\de\salida"
```

Dentro de esta ruta se crea automáticamente una carpeta para cada chip:

```text
F5C9_aging/
F4C4_aging/
...
```

## Ejecución

El programa se puede ejecutar desde la terminal mediante:

```bash
python GRATMA_measures.py
```

Al comenzar solicita:

1. Número de equipos que se van a medir.
2. Puerto COM de cada equipo.
3. Nombre del wafer.
4. Nombre del chip.

Por ejemplo:

```text
Número de equipos a medir en paralelo: 2

--- Equipo 1/2 ---
Puerto COM: COM4
Wafer: APS5
Chip: F4C4

--- Equipo 2/2 ---
Puerto COM: COM7
Wafer: APS5
Chip: F5C9
```

## Funcionamiento de la medida

Una vez iniciado el programa:

1. Se crean las carpetas de salida.
2. Se abren los puertos serie.
3. Se configura el estado eléctrico mediante los comandos `um`, `sw` y `sv`.
4. Se realiza el tiempo de estabilización inicial.
5. Cada GRATMA comienza las medidas de forma independiente.
6. Los 8 sensores se miden siguiendo un orden aleatorio.
7. Se ejecuta el barrido I-V mediante el comando `iv`.
8. Se extraen y validan los valores `Vfg`, `Vs`, `Ig` e `Is`.
9. Solo las medidas válidas se guardan como TXT final.

El orden de los sensores cambia en cada secuencia. Los sensores 1–4 y 5–8 se aleatorizan por separado y se alternan durante la medida para evitar medir siempre en el mismo orden.

## Archivos generados

Los archivos finales siguen el formato:

```text
Wafer_Chip_stage_ArrayN_random_Secuencia_Electrolito.txt
```

Por ejemplo:

```text
APS5_F5C9_stage_Array3_random_2_PB-S0_01.txt
```

Al comienzo de cada archivo se guarda información sobre la configuración utilizada durante la medida.

Los datos se guardan con las columnas:

```text
Vfg;Vs;Ig;Is
```

Los valores de `Vfg`, `Vs`, `Ig` e `Is` se obtienen directamente de la información devuelta por el GRATMA.

También se genera un archivo:

```text
All_info_...
```

que contiene toda la información recibida por el puerto serie y permite revisar detalladamente la medida.

## Control de errores

El programa incluye diferentes comprobaciones para evitar guardar medidas incorrectas.

Una medida puede detenerse si ocurre alguno de estos problemas:

- Error de comunicación con el puerto serie.
- Tiempo máximo de medida superado.
- Ausencia prolongada de datos.
- Error explícito devuelto por el firmware.
- Número incorrecto de puntos recibidos.
- Valores no válidos (`NaN` o infinito).
- Error durante la inicialización o configuración del GRATMA.

Si una medida falla, no se genera el TXT final y el archivo `All_info_` se conserva como:

```text
All_info_....FAILED.txt
```

### Recuperación automática

Si el firmware devuelve:

```text
Cannot start sweep - system not ready (state=4)
```

el programa envía automáticamente:

```text
reset
```

y vuelve a intentar una vez la medida del mismo sensor.

Si el problema continúa después del reset, ese GRATMA se detiene.

Un fallo en un GRATMA **no detiene los demás equipos** que estén midiendo en paralelo. Solo `Ctrl+C` provoca una parada global.

## Protección de archivos

Por defecto:

```python
ALLOW_OVERWRITE = False
```

Esto evita sobrescribir accidentalmente una medida que ya existe.

Si el programa detecta un TXT final existente para ese chip, no comienza una nueva medida sobre esos archivos.

## Medidas en paralelo

Cada GRATMA se ejecuta en un hilo diferente, por lo que varios dispositivos conectados a diferentes puertos COM pueden medir al mismo tiempo.

Los mensajes de terminal aparecen identificados mediante su puerto:

```text
[COM4] ...
[COM7] ...
```

Esto permite seguir de forma independiente el estado de cada equipo.

El programa no permite utilizar dos veces el mismo puerto COM ni asignar el mismo nombre de chip a dos equipos diferentes.

## Notas

- Comprobar los puertos COM antes de comenzar.
- Modificar `FOLDER_PATH` según el ordenador utilizado.
- Comprobar que ningún otro programa esté utilizando los puertos serie.
- Revisar los archivos `All_info_` si una medida presenta algún problema.
- Los archivos `.FAILED.txt` indican medidas que no se han considerado válidas.
- Al finalizar se muestra un resumen con el número de medidas válidas y el estado de cada GRATMA.
