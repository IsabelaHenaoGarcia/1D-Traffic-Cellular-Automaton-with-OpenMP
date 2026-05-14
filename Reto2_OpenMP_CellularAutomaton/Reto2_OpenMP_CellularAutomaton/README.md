# Reto 2 — Cellular Automaton de tráfico con OpenMP

Implementación serial + paralela (OpenMP) del autómata celular 1D de
tráfico descrito por ARCHER/EPCC (`trafficmodel.pdf`). Incluye scripts
listos para correr todos los experimentos: barrido velocidad-densidad,
strong scaling con varios schedulers, análisis de chunk size, comparación
contra anti-patrones (`critical`, `atomic`, false sharing) y profiling
con gprof y `/usr/bin/time`.

> Si solo quieres **leer los resultados y el análisis**, abre
> [`INFORME.md`](INFORME.md). Este README es para **reproducir** los
> experimentos en tu propia máquina.

---

## 1. Requisitos

### 1.1 Sistema operativo

Cualquier Linux moderno (probado en Ubuntu 24.04 sobre WSL2 con kernel
6.6). Debería funcionar también en macOS con `clang` y `libomp`.

### 1.2 Paquetes

| Paquete             | Mínimo                | Uso                              |
| ------------------- | --------------------- | -------------------------------- |
| `gcc`               | 9.0 (recomendado 11+) | Compilar (soporta OpenMP 4.5)    |
| `make`              | cualquiera            | Build                            |
| `python3`           | 3.8+                  | Generar gráficos                 |
| `python3-matplotlib`| 3.x                   | Gráficos PNG                     |
| `binutils`          | (trae `gprof`)        | Perfilado de funciones           |
| `time`              | GNU `/usr/bin/time`   | Estadísticas de SO (RSS, CPU%)   |
| `perf`              | opcional              | Contadores HW (no funciona WSL2) |

**Instalación en Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install -y build-essential python3 python3-matplotlib time \
                    linux-tools-generic
```

**Instalación en Fedora/RHEL:**

```bash
sudo dnf install -y gcc make python3 python3-matplotlib time perf
```

**Instalación en macOS (con Homebrew):**

```bash
brew install gcc libomp python matplotlib
# usa 'gcc-13' en lugar de 'gcc' en el Makefile, o:
# make CC=gcc-13
```

### 1.3 Hardware recomendado

- Procesador con **al menos 2 cores físicos** (la paralelización satura
  alrededor de 4 hilos por ser memory-bound — ver `INFORME.md` §6).
- **8 GB de RAM** son suficientes (los arrays más grandes del default
  ocupan ~160 MB).
- Hasta **12 hilos** en el script de scaling. Si tu CPU tiene menos
  hilos, edita la variable `THREADS_LIST` (sección 5).

---

## 2. Estructura del proyecto

```
.
├── Makefile
├── README.md                # esta guia
├── INFORME.md               # analisis tecnico completo (con plots)
├── trafficmodel.pdf         # enunciado original (ARCHER/EPCC)
│
├── src/                     # codigo fuente
│   ├── common.h             # opciones CLI, RNG xorshift, timers
│   ├── traffic_serial.c     # version serial de referencia
│   ├── traffic_omp.c        # version paralela (varios schedulers)
│   └── traffic_omp_bad.c    # versiones critical / atomic / false-share
│
├── scripts/                 # experimentos automatizados
│   ├── run_velocity_density.sh
│   ├── run_scaling.sh
│   ├── run_chunk_sweep.sh
│   ├── run_antipatterns.sh
│   ├── run_profiling.sh
│   └── plot_results.py
│
├── bin/                     # binarios compilados (creado por make)
├── results/                 # CSVs y profile (creado por scripts)
└── plots/                   # graficos PNG (creado por plot_results.py)
```

---

## 3. Compilación

Un solo paso compila los 5 binarios:

```bash
make all
```

Salida esperada:

```
bin/traffic_serial          # version serial optimizada (-O3 -march=native)
bin/traffic_omp             # version OpenMP optimizada
bin/traffic_omp_bad         # versiones con anti-patrones (didactico)
bin/traffic_serial_prof     # version serial con -pg para gprof
bin/traffic_omp_prof        # version OMP con -pg para gprof
```

Si tu CPU no soporta `-march=native`, edita la variable `OPT` en el
[`Makefile`](Makefile) y cambia a `-march=x86-64-v2` o similar.

**Otros targets:**

```bash
make clean        # elimina bin/
make distclean    # elimina bin/, results/*.csv, plots/*.png
```

---

## 4. Uso rápido del binario

```bash
./bin/traffic_serial -n 2000000 -s 200 -d 0.5 -r 42 -v
./bin/traffic_omp    -n 2000000 -s 200 -d 0.5 -r 42 -t 8 -S static -v
```

Salida típica (modo `-v`):

```
[serial] N=2000000 steps=200 density=0.500 -> actual=0.4996 \
         ncars0=999283 ncars1=999283 v_asymp=0.97189 \
         time=0.2476s (1615.40 MCUPS)
```

### 4.1 Argumentos

| Flag           | Default     | Descripción                                  |
| -------------- | ----------- | -------------------------------------------- |
| `-n N`         | `1000000`   | Tamaño de la carretera (celdas)              |
| `-s steps`     | `200`       | Número de pasos de tiempo                    |
| `-d density`   | `0.5`       | Densidad objetivo (0.0–1.0)                  |
| `-r seed`      | `42`        | Semilla del RNG (reproducibilidad)           |
| `-t threads`   | `omp_max`   | Hilos OpenMP (solo `traffic_omp*`)           |
| `-S sched`     | `static`    | `static` / `dynamic` / `guided` / `auto`     |
| `-c chunk`     | `0`         | Chunk size (0 = default OpenMP)              |
| `-o file.csv`  | —           | Guarda métricas en CSV (append)              |
| `--tag str`    | `run`       | Etiqueta de la corrida en el CSV             |
| `-v`           | off         | Salida detallada en stderr                   |
| `-h`           | —           | Ayuda                                        |

### 4.2 Validaciones rápidas

El programa retorna `exit_code != 0` si NO se conserva el número de
carros (debe permanecer constante en cada paso, propiedad del modelo):

```bash
./bin/traffic_omp -n 10000000 -s 1000 -d 0.6 -t 8 && echo "OK conservacion"
```

Pruebas de los casos límite del PDF:

```bash
./bin/traffic_serial -n 10000 -s 100 -d 1.0 -v   # esperado: v_asymp=0.00
./bin/traffic_serial -n 10000 -s 500 -d 0.01 -v  # esperado: v_asymp=1.00
```

---

## 5. Ejecutar los experimentos completos

Hay un script por tipo de experimento. Todos van escribiendo CSVs en
`results/` para que `plot_results.py` los grafique.

> **Tiempo total estimado** (en una CPU como Ryzen 5 3600):
> velocity_density ≈ 1 min, scaling ≈ **15–20 min** (dominado por
> `dynamic`, que es muy lento a propósito), chunk_sweep ≈ 8 min,
> antipatterns ≈ 30 s, profiling ≈ 5 s. Hay variables de entorno para
> reducirlos.

### 5.1 Curva velocidad vs densidad

```bash
./scripts/run_velocity_density.sh
```

Variables opcionales:

```bash
N=1000000 STEPS=300 THREADS=4 ./scripts/run_velocity_density.sh
```

Produce `results/velocity_density.csv`.

### 5.2 Strong scaling

```bash
./scripts/run_scaling.sh
```

Variables opcionales (útiles para CPUs con pocos hilos):

```bash
N=5000000 STEPS=50 REPS=2 THREADS_LIST="1 2 4 8" \
SCHEDULERS="static guided" \
./scripts/run_scaling.sh
```

> **Aviso:** la opción `dynamic` con `chunk=1` (default) es **50× más
> lenta** que `static` — esto es intencional para mostrar el anti-patrón.
> Si quieres acelerar la suite, exporta `SCHEDULERS="static guided"`.

Produce `results/scaling.csv`.

### 5.3 Chunk sweep

```bash
./scripts/run_chunk_sweep.sh
```

Mismo aviso: cubre `dynamic` con varios chunks → tarda.

Produce `results/chunk_sweep.csv`.

### 5.4 Anti-patrones (`critical`, `atomic`, false sharing)

```bash
./scripts/run_antipatterns.sh
```

Rápido (~30 s). Demuestra cuantitativamente por qué se usa `reduction`.

Produce `results/antipatterns.csv`.

### 5.5 Profiling

```bash
./scripts/run_profiling.sh
```

Genera en `results/profile/`:

- `gprof_serial.txt`, `gprof_omp_t1.txt` — perfil de funciones.
- `time_serial.txt`, `time_omp_t8.txt` — `/usr/bin/time -v` con RSS,
  page faults, % CPU.
- `bandwidth.txt` — cálculo de ancho de banda efectivo desde
  `scaling.csv`.

> Si tienes `perf` funcional (no WSL2), el script lo intenta pero
> no falla si no está disponible.

### 5.6 Todo de una vez

```bash
make all
./scripts/run_velocity_density.sh
./scripts/run_scaling.sh
./scripts/run_chunk_sweep.sh
./scripts/run_antipatterns.sh
./scripts/run_profiling.sh
python3 scripts/plot_results.py
```

---

## 6. Generar los gráficos

```bash
python3 scripts/plot_results.py
```

Produce en `plots/`:

| Archivo                     | Contenido                                     |
| --------------------------- | --------------------------------------------- |
| `velocity_vs_density.png`   | Curva clásica de la regla 184                 |
| `scaling.png`               | Tiempo y speedup vs hilos por scheduler       |
| `efficiency.png`            | Eficiencia paralela vs hilos                  |
| `chunk_sweep.png`           | Tiempo vs chunk size en static / dynamic      |
| `antipatterns.png`          | Comparación de barras de los anti-patrones    |

El script solo grafica los CSVs que existen, así que es seguro correrlo
tras ejecutar solo un subconjunto de experimentos.

---

## 7. Personalización

### 7.1 Cambiar el tamaño del problema

Edita las variables al inicio de cada script bash, o expórtalas:

```bash
N=50000000 STEPS=200 ./scripts/run_scaling.sh
```

Cuidado: `N=50M` con `int` = 200 MB por array → necesitas ~500 MB libres.

### 7.2 Probar otra densidad

```bash
DENS=0.7 ./scripts/run_scaling.sh
```

Para `ρ > 0.5`, la velocidad asintótica es menor que 1 y el sistema
necesita más pasos para converger (recomendado `STEPS≥400`).

### 7.3 Otro compilador

```bash
make CC=clang OMPFLAG=-fopenmp
make CC=gcc-13               # macOS con Homebrew
```

### 7.4 Cambiar el tipo de celda

En `src/common.h:11`:

```c
typedef int cell_t;          // por defecto
```

Cambiar a `char` reduce el footprint de memoria 4× y, dado que el kernel
es *memory-bound* (ver `INFORME.md` §6), puede mejorar el throughput
hasta 4×. Es la mejora natural para escalar más allá del ancho de banda.

---

## 8. Solución de problemas comunes

### Error: `cannot find -fopenmp`

Tu GCC no fue compilado con soporte OpenMP. En Ubuntu:

```bash
sudo apt install libgomp1
```

### `dynamic` es extremadamente lento (>1 min por corrida)

**Es esperado.** Es uno de los puntos del análisis: `dynamic` con chunk
pequeño es 50–100× más lento que `static` para este kernel. Para
acelerar las pruebas, omite `dynamic`:

```bash
SCHEDULERS="static guided" ./scripts/run_scaling.sh
```

### `perf: command not found` o `WARNING: perf not found for kernel`

`perf` no está disponible (común en WSL2). El script lo detecta y usa
`/usr/bin/time -v` y gprof como alternativa — los CSVs se generan
igual. El cálculo de ancho de banda efectivo lo hace `plot_results.py`
desde los datos del binario directamente.

### Los plots salen vacíos / con menos curvas

Solo se grafica lo que esté en los CSVs. Si quieres todas las curvas,
corre todos los scripts antes de `plot_results.py`.

### En macOS, "ld: library not found for -lomp"

Necesitas `libomp`:

```bash
brew install libomp
make CC=clang OMPFLAG="-Xpreprocessor -fopenmp -lomp"
```

---

## 9. Reproducibilidad

Todas las corridas son **deterministas** dada la misma semilla `-r` y
mismo `N` (los hilos solo computan, no afectan el contenido). Verifica:

```bash
./bin/traffic_serial -n 1000000 -d 0.5 -r 7 -v | grep v_asymp
./bin/traffic_omp    -n 1000000 -d 0.5 -r 7 -t 8 -v | grep v_asymp
# Misma v_asymp en ambos.
```

Si dos corridas con misma semilla difieren ⇒ hay un bug de paralelismo
(probablemente data race). La versión actual del repo está validada
contra esta propiedad.

---

## 10. Para más detalle

- **Análisis completo** (decisiones, resultados, causas raíz, roofline,
  conclusiones): [`INFORME.md`](INFORME.md).
- **Enunciado original**: [`trafficmodel.pdf`](trafficmodel.pdf).
- **Material de referencia ARCHER/EPCC:**
  <http://www.archer.ac.uk/training/course-material/2019/07/mpi-epcc/index.php>

---

## 11. Licencia

Material de estudio derivado del curso público de ARCHER/EPCC. Uso
académico permitido. El código de este repo es de libre uso para fines
educativos.
