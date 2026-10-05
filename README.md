# Optimización del portafolio de variedades de rosa

Asignación de superficie de invernadero entre variedades de rosa para maximizar el margen
por metro cuadrado, con riesgo controlado y restricciones operativas de cultivo perenne.

Proyecto CAPSTONE — Maestría en Inteligencia de Negocios y Ciencia de Datos
**Bryan Steve Vargas Cabascango**

---

## Qué hace este proyecto

Una florícola exportadora decide qué variedades sembrar comparando productividad promedio
(tallos por metro cuadrado). El proyecto demuestra que ese criterio ignora la mitad de los
determinantes de la rentabilidad y cuantifica cuánto cuesta.

El análisis adapta la teoría del portafolio de Markowitz, aplicada por
[Barkley et al. (2010)](https://doi.org/10.1017/S107407080000328X) a variedades de trigo,
con tres modificaciones: se optimiza el **margen económico por m²** en lugar del rendimiento
físico, el riesgo se estima por **encogimiento de Ledoit-Wolf**, y se incorporan
**restricciones de crecimiento y rotación de camas**.

### Resultados principales

| Indicador | Valor |
|---|---|
| Variedades con margen medio negativo | 39 de 102 (25,4 % de la superficie) |
| De ellas, negativas con significancia estadística | 18 (11,9 % de la superficie) |
| Costo de oportunidad de la asignación vigente | **482.172 USD/año** |
| Intervalo de confianza al 95 % (bootstrap por bloques) | 353.841 – 501.228 USD/año |
| Núcleo de decisiones estadísticamente robustas | 14 eliminaciones + 11 ampliaciones |
| Ganancia del núcleo robusto | 312.899 USD/año rotando el 10,7 % del área |
| Mejora del modelo predictivo sobre la media histórica | 9,8 % en RMSE (Diebold-Mariano, p = 0,004) |

---

## ⚠️ Los datos no están incluidos

**Las tres bases de datos son información confidencial de la empresa y no se publican en
este repositorio.** Contienen producción, ventas, precios y costos reales de una operación
comercial activa.

Para replicar el análisis se necesitan tres archivos Excel con la estructura que se detalla
a continuación, colocados en una carpeta `datos/` en la raíz del repositorio:

```
datos/
├── dbo_panel_tesis_db_v1_20260913.xlsx   (panel de producción y ventas)
├── dbo_crop_plan_clean_mo.xlsx           (plan de cultivo)
└── dbo_costs_v1.xlsx                     (estado de costos)
```

Si los nombres de archivo difieren, se ajustan las constantes `ARCH_PANEL`, `ARCH_PLAN` y
`ARCH_COSTOS` en la primera celda del cuaderno `01_EDA_Portafolio_Rosas.ipynb`.

---

## Diccionario de variables

### 1. Panel de producción y ventas

Periodicidad mensual, nivel **variedad × mes**. En el estudio original: 10.327 filas,
223 variedades, 80 meses (enero 2020 – agosto 2026).

| Columna | Tipo | Descripción |
|---|---|---|
| `variety` | texto | Nombre comercial de la variedad de rosa. Clave de unión con el plan de cultivo. |
| `id_month` | fecha | Primer día del mes (`YYYY-MM-01`). Clave temporal de unión. |
| `gross_prod` | entero | Producción bruta de tallos cosechados en el mes. |
| `exportable` | entero | Tallos que cumplen calidad de exportación. |
| `national` | entero | Tallos de calidad nacional. En esta operación se descartan íntegramente. |
| `sales` | entero | Tallos efectivamente vendidos en el mes. |
| `discarted` | entero | Tallos dados de baja. |
| `%_discarted` | decimal | Proporción de tallos dados de baja sobre la producción bruta (0 a 1). |
| `avg_price` | decimal | Precio promedio por tallo vendido, en USD. |
| `sales_usd` | decimal | Ventas del mes en USD. |
| `year` | entero | Año calendario. Redundante con `id_month`. |
| `season_id` | texto | Temporada comercial: `REGULAR`, `VALENTIN`, `MADRES`, `DIFUNTOS`, `NAVIDAD`. |
| ``is_valentines_season` `` | binario | Indicador de temporada de San Valentín. **El nombre incluye una comilla invertida final**; los cuadernos la renombran a `is_valentin`. |
| `is_mothers_day_season` | binario | Indicador de temporada del Día de la Madre. |
| `raw_is_mothers_day_season` | binario | Versión sin depurar del indicador anterior. No se utiliza. |
| `is_all_saints_season` | binario | Indicador de temporada de Difuntos. |
| `is_christmas_season` | binario | Indicador de temporada de Navidad. |
| `is_covid_shock` | binario | Meses de interrupción comercial de 2020. |
| `is_censored` | binario | La variedad colocó toda su producción exportable en el mes. No se usa en el modelo final; queda disponible para estimar demanda. |

### 2. Plan de cultivo

Periodicidad mensual, nivel **variedad × mes**. En el estudio original: 10.399 filas,
218 variedades.

| Columna | Tipo | Descripción |
|---|---|---|
| `clean_variety.1` | texto | Nombre de la variedad. **El nombre de la columna incluye el sufijo `.1`**; los cuadernos la renombran a `variety`. |
| `id_month` | fecha | Primer día del mes. |
| `year` | entero | Año calendario. Contiene un error de etiquetado en enero de 2025 que genera 127 pares variedad-mes duplicados; los cuadernos los eliminan conservando la primera ocurrencia. |
| `n_plants` | decimal | Número de plantas sembradas de la variedad. Inductor del costo de campo. |
| `area` | decimal | Superficie productiva asignada, en m². Denominador del margen. |
| `density` | decimal | Plantas por m² (aproximadamente 5,6 a 9,1). |
| `monthly_flowers_per_plant` | decimal | Producción mensual esperada por planta (0,6 a 2,0). Se usa en el control de consistencia física. |

### 3. Estado de costos

Periodicidad mensual, nivel **finca**. En el estudio original: 80 filas, una por mes.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_month` | fecha | Primer día del mes. |
| `year`, `month_num` | entero | Año y número de mes. Redundantes con `id_month`. |
| `total_cost` | decimal | Costo total mensual de la finca, en USD. Igual a `farm_cost` + `pharvest_cost`. |
| `farm_cost` | decimal | Costo de campo: fertilización, riego, mano de obra, control fitosanitario. Se imputa por plantas-mes. |
| `pharvest_cost` | decimal | Costo de poscosecha: clasificación, boncheo y empaque. Se imputa por tallos procesados. |
| `gross_prod`, `exportable`, `national`, `sales`, `discarted` | entero | Totales de la finca en el mes. Se usan para verificar la reconciliación. |
| `sales_usd`, `price` | decimal | Ventas totales y precio promedio de la finca. |
| `n_plants`, `area`, `density` | decimal | Totales de la finca. |

> **Verificación obligatoria:** los cuadernos comprueban que la suma de los costos imputados
> por variedad reproduzca `total_cost` de cada mes. El cuaderno 1 contiene un `assert` que
> detiene la ejecución si la diferencia supera 10⁻⁶ %. Si falla, el problema está en los
> datos de entrada, no en el código.

---

## Estructura del repositorio

```
.
├── README.md
├── requirements.txt
├── .gitignore
├── datos/                              ← vacío; aquí van los tres Excel (no versionados)
├── 01_EDA_Portafolio_Rosas.ipynb
├── 02_Modelado_Predictivo_Prescriptivo.ipynb
├── 03_Optimizacion_Hiperparametros.ipynb
├── 04_Resultados_Estrategia.ipynb
├── figuras/                            ← salida del cuaderno 1 (no versionada)
├── modelo/                             ← salida de los cuadernos 2 y 3 (no versionada)
└── resultados/                         ← salida del cuaderno 4 (no versionada)
```

---

## Requisitos

Python 3.12. Todas las librerías son de código abierto:

```bash
pip install -r requirements.txt
```

```
pandas>=2.0
numpy>=1.24
matplotlib>=3.7
scikit-learn>=1.3
scipy>=1.10
openpyxl>=3.1
jupyter
```

---

## Orden de ejecución

Los cuadernos **se ejecutan en orden**, porque cada uno consume las salidas del anterior.

| N.º | Cuaderno | Entrada | Salida | Duración |
|---|---|---|---|---|
| 1 | `01_EDA_Portafolio_Rosas` | Los tres Excel | `figuras/panel_analitico.pkl` + 19 figuras | ~1 min |
| 2 | `02_Modelado_Predictivo_Prescriptivo` | Panel analítico | `modelo/residuos.pkl`, `modelo/asignacion.csv` + 13 figuras | ~3 min |
| 3 | `03_Optimizacion_Hiperparametros` | Panel analítico y asignación | `modelo/tuning_rf.csv` + 1 figura | ~6 min |
| 4 | `04_Resultados_Estrategia` | Panel, residuos y asignación | `resultados/inferencia.json` + 10 figuras | ~4 min |

### Qué hace cada cuaderno

**1. Análisis exploratorio.** Integra las tres fuentes, depura duplicados e inconsistencias
físicas, imputa los costos por variedad y verifica la reconciliación contable. Construye la
matriz de descripción de variables, la matriz de correlación y 19 visualizaciones. Deja el
panel analítico depurado (102 variedades, 7.509 observaciones).

**2. Modelado predictivo y prescriptivo.** Compara Random Forest, Extra Trees y Gradient
Boosting contra dos líneas base, con partición temporal y validación de ventana expansiva.
Estima la matriz de riesgo con encogimiento de Ledoit-Wolf y resuelve el programa cuadrático
de media-varianza con restricciones de crecimiento y rotación. Produce la frontera de
eficiencia y la asignación óptima.

**3. Optimización de hiperparámetros.** Búsqueda en rejilla de 32 combinaciones con
validación cruzada temporal de cuatro pliegues, regla de un error estándar y comprobación de
que la asignación recomendada es invariante a la configuración elegida.

**4. Resultados, inferencia y estrategia.** Prueba de Diebold-Mariano, significancia del
margen por variedad con errores de Newey-West, bootstrap por bloques de 500 réplicas,
clasificación de la estabilidad de cada decisión, escenarios de ejecución, viabilidad
económica y análisis de sensibilidad a los parámetros operativos.

---

## Notas de reproducibilidad

- **Semillas fijas.** Todos los modelos usan `random_state=42` y el bootstrap
  `np.random.default_rng(42)`. Con los mismos datos y versiones, los resultados son idénticos.
- **Convergencia del optimizador.** SLSQP puede no converger con tolerancia estricta según la
  versión de SciPy. La función `resolver()` reintenta con tolerancias de `1e-10`, `1e-8` y
  `1e-7` y dos puntos de partida. Si aun así falla, un `assert` lo informa en lugar de
  propagar un error oscuro.
- **Bootstrap.** De 500 réplicas, alrededor de 499 convergen. Las que no se descartan, y el
  cuaderno reporta cuántas fueron.
- **Versiones verificadas.** El repositorio se ejecutó de extremo a extremo con Python 3.12,
  pandas 3.0, NumPy 2.4, scikit-learn 1.8, SciPy 1.17 y Matplotlib 3.10, reproduciendo las
  cifras reportadas (costo de oportunidad de 482.172 USD, reconciliación de 2,22 × 10⁻¹⁴ %).
- **Parámetros de negocio.** En el cuaderno 4 se ajustan `CAP` (participación máxima por
  variedad, 0,25), `CRECE` (crecimiento máximo por ciclo, 2,0) y `TUR` (rotación anual máxima,
  0,20). El análisis de sensibilidad de la sección 9 muestra cómo cambia el resultado con
  otros valores.

---

## Limitaciones conocidas

1. El margen por m² depende de la regla de imputación de costos; una regla alternativa
   produciría márgenes distintos.
2. El modelo estima el margen condicionado a la escala actual de cada variedad y luego
   recomienda cambiarla. La restricción de crecimiento acota el riesgo pero no lo elimina.
3. La demanda por variedad no se estima de forma explícita: las ventas de una variedad que se
   coloca íntegramente son una cota inferior de su demanda, no su techo.
4. La medida de riesgo es simétrica. Contrastar con media-déficit esperado queda pendiente.

---

## Cita

Vargas Cabascango, B. S. (2026). *Optimización del portafolio de variedades y la asignación
de superficie para maximizar el margen por metro cuadrado: el caso de una florícola
exportadora de rosas* [Proyecto CAPSTONE, Maestría en Inteligencia de Negocios y Ciencia de
Datos].

El archivo `Bibliografia_Capstone.ris` contiene las 35 referencias del proyecto en formato
importable a Zotero, Mendeley o EndNote.

