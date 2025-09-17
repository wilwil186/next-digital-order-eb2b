
# Predicción del **Próximo Pedido DIGITAL** (eB2B)

**Autor:** Wilson Eduardo Jerez Hernández  
**Fecha:** 2025-09-16

---

## 🧾 TL;DR (1 minuto)

- **Objetivo:** estimar la probabilidad de que el **próximo pedido** de un cliente sea **DIGITAL** para **priorizar campañas**.  
- **Enfoque:** features temporales a nivel **cliente-mes** + **Regresión Logística** (PySpark) con manejo de **desbalance** y **backtesting** por cortes de tiempo.  
- **Valor de negocio:** en el **top-10%** de clientes, el modelo ofrece **Lift ≈ 1.55×** (duplica la tasa vs. al azar) con precisión entre **~48–58%**.  
- **Listo para explicar:** este README contiene **Qué / Por qué / Cómo**, mini-teoría de **Backtesting**, **AUC ROC/PR**, **PR@k/Lift**, guía de **demo** y **FAQs**.

---

## 1) Problema & Objetivo

La compañía quiere **acelerar la adopción del canal digital** (eB2B). Con el histórico de pedidos por cliente, construimos un modelo que **rankea** clientes por su probabilidad de que **el próximo pedido** sea **DIGITAL**.

**Objetivo operativo:** seleccionar **top-k%** clientes (según score) para **campañas** con mayor conversión esperada.  

> **Por qué “próximo” y no “actual”**: el equipo de marketing decide **hoy** a quién contactar; por lo tanto, la predicción debe mirar **solo el pasado** y anticipar el **siguiente** comportamiento.

---

## 2) Datos & Preparación (Qué / Por qué / Cómo)

**Nivel de trabajo:** `cliente_id × mes` (columna `ym`).

**Campos mínimos esperados (ejemplo):**  
`cliente_id, pais_cd, fecha_pedido_dt, canal_pedido_cd, facturacion_usd_val, materiales_distintos_val, cajas_fisicas, frecuencia_visitas_cd, ...`

**Normalización temporal (Cómo):**
- `month_first = trunc(fecha_pedido_dt, 'month')`
- `ym = date_format(month_first, 'yyyy-MM')`
- `is_digital = 1{canal_pedido_cd == 'DIGITAL'} else 0`

**¿Por qué a cliente-mes?**  
- Facilita **features temporales** (rolling/recencia).  
- Evita mezclar observaciones de distinta granularidad.  
- Alinea la **ventana de decisión** con el uso real (campañas mensuales o quincenales).

---

## 3) Etiquetado **sin fuga** (Qué / Por qué / Cómo)

- **Qué:** por cada `cliente_id` y mes, tomamos el **último pedido** del mes.  
  La **etiqueta** (`label`) vale **1** si el **siguiente pedido** del cliente fue **DIGITAL** (si no, **0**).
- **Por qué:** queremos saber si **el próximo** pedido será digital; usar información del mismo mes para etiquetar introduciría **fuga** (*leakage*).
- **Cómo:**  
  - Ventana ordenada por `fecha_pedido_dt`: `lead(canal_pedido_cd)` da el canal del **siguiente pedido**.  
  - Dentro del mes, `row_number()` descendente para quedarnos con **el último** pedido del mes.  
  - La recencia se calcula con `lag(fecha_pedido_dt)` → `recency_days_last`.

---

## 4) Features (Qué / Por qué / Cómo)

**Grupos de señales**:
1. **Actividad y valor (RFM light):**  
   - `n_orders`, `sum_fact`, `avg_fact`, `sum_cajas`, `avg_cajas`, `avg_mat_dist`.
2. **Comportamiento digital propio:**  
   - `digital_ratio` (del mes), `lag1_digital_ratio`, `digital_ratio_3m`, `growth_digital_ratio`.
3. **Ciclo de vida:**  
   - `months_since_first` (meses desde el primer pedido del cliente).
4. **Contexto del segmento (priors):**  
   - `region_digital_ratio_lag1`, `tipo_digital_ratio_lag1` (lag del ratio DIGITAL por **región** y **tipo_cliente**).
5. **Categóricas de baja cardinalidad:**  
   - `madurez_digital_cd`, `frecuencia_visitas_cd`, `pais_cd`, `tipo_cliente_cd` → **OHE**.

**Por qué estos features:** combinan **señales propias** del cliente, **tendencias** recientes y **contexto** de su segmento, claves para **ranking**.

---

## 5) Modelo & Entrenamiento

- **Modelo base:** **Regresión Logística** (PySpark) por **estabilidad, velocidad y explicabilidad**.  
- **Desbalance de clases:** la clase `1 = DIGITAL` suele ser minoritaria. Se implementan 3 estrategias:  
  1) `weightCol` (por defecto), 2) **oversampling** de positivos, 3) **undersampling** de negativos.  
- **Grid sencillo:** `regParam ∈ {0, 0.01, 0.1}`, `elasticNetParam ∈ {0, 0.5, 1}` con `TrainValidationSplit`.  
- **Métrica de tuning:** **AUC PR** (más sensible a desbalance que AUC ROC).

> **Explicabilidad**: con LR se pueden inspeccionar **coeficientes** (signo/magnitud). Útil para insights de negocio (con precaución por colinealidad).

---

## 6) Validación Temporal (**Backtesting**)

**Qué**: simular el desempeño del modelo **en el pasado** como si lo hubiéramos desplegado allí.  
**Cómo**: para cada corte `TEST_START_YM`, entrenar con `ym < corte` y evaluar en `ym ≥ corte`.  
**Por qué**: los datos cambian; el backtesting prueba **robustez temporal**.

**Cortes usados:** `2023-08`, `2023-10`, `2023-12`, `2024-01`  
**Entorno:** Spark **4.0.1**.  
**Tamaño agregado (cliente-mes):** ~**1,022,849** filas.

---

## 7) Métricas & Resultados

### Mini-teoría
- **AUC ROC**: área bajo la curva TPR vs FPR; **separación global** (1.0 perfecto, 0.5 azar).  
- **AUC PR**: área bajo Precisión vs Recall; **más informativa** en **desbalance**.  
- **PR@k**: precisión en el **top k%** por score.  
- **Recall@k**: fracción de positivos capturada en ese **top k%**.  
- **Lift@k = PR@k / tasa_base**: cuántas veces mejor que al azar.  
- **F1**: 2·(Precision·Recall)/(Precision+Recall) — se reporta al **umbral óptimo**.

### Backtesting (estrategia `weights`)
| Corte | AUC ROC | AUC PR | Umbral F1 | F1@Umbral | PR@5% | Lift@5% | PR@10% | Lift@10% |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 2023-08 | 0.6233 | 0.4997 | 0.00 | 0.5422 | 0.5796 | 1.5582 | 0.5790 | 1.5566 |
| 2023-10 | 0.6196 | 0.4735 | 0.38 | 0.5236 | 0.5492 | 1.5578 | 0.5478 | 1.5539 |
| 2023-12 | 0.6146 | 0.4375 | 0.38 | 0.4993 | 0.5100 | 1.5651 | 0.5067 | 1.5548 |
| 2024-01 | 0.6119 | 0.4135 | 0.38 | 0.4823 | 0.4829 | 1.5697 | 0.4791 | 1.5571 |

**Lecturas clave**  
- **Estabilidad**: rendimiento **consistente** entre cortes (ligera degradación esperable).  
- **Valor para campañas**: **Lift ≈ 1.55×** en top-5–10% → **duplica** la efectividad frente a contactar al azar.  
- **Precisión top-k**: ~**48–58%** en top-10% (según corte).  
- **AUC PR** disminuye en cortes recientes → posible **drift**; sugiere **reentrenos**.

---

## 8) Hallazgos

1) El enfoque **ranker** (probabilidad → top-k) es **útil** para priorizar clientes en campañas con presupuesto limitado.  
2) **Señales económicas** (p. ej., `avg_fact_imp`) y **tendencias** (`digital_ratio_3m`, `growth_digital_ratio`) aportan a la discriminación.  
3) Los **priors de segmento** (región/tipo) ayudan a capturar patrones contextuales.  
4) La **optimización por AUC PR** y el reporte de **PR@k/Lift** son mejores para **desbalance** que accuracy.

---

## 9) Cómo ejecutar (reproducibilidad)

### Requisitos
- Python 3.10+  
- PySpark (probado en Spark 4.0.1)

```bash
# instalar dependencias
pip install -r requirements.txt
```

### Datos
- Coloca los archivos en `dataset/` (por defecto en los notebooks).  
- Si tu ruta cambia, edita `DATA_DIR` al inicio del notebook de modelado.

### Pasos
1) Abre y corre **EDA.ipynb** para validar esquema/distribuciones.  
2) Abre y corre **02_Modelado** (notebook):  
   - Ajusta `BACKTEST_SPLITS` si quieres otros cortes.  
   - Para pruebas rápidas: baja `spark.sql.shuffle.partitions` y/o reduce el **grid** del modelo.  
3) (Opcional) Genera **scores** y exporta **top-k** clientes para campaña.

> **Estabilidad**: el notebook usa **Kryo**, incrementa **memoria**, y aplica `checkpoint(eager=True)` para cortar linajes de Spark y evitar OOM.

---

## 10) Uso en campañas (cómo accionar)

1) Ordena clientes por `p1` (probabilidad de pedido DIGITAL).  
2) Selecciona **top-k%** según tu presupuesto/contactabilidad.  
3) Usa **PR@k** y **Lift@k** para estimar **impacto** (conversión esperada y ROI).  
4) Ajusta el **umbral** según tu objetivo:  
   - **Mayor eficiencia** → sube el umbral (más precisión, menos cobertura).  
   - **Mayor cobertura** → bájalo (más recall, menor precisión).

---

## 11) Limitaciones

- **Modelo lineal** (LR): robusto y explicable, pero puede no capturar **no linealidades** complejas.  
- **Drift temporal**: caída leve de AUC PR en cortes recientes; requiere **monitoreo** y **reentrenos**.  
- **Features mejorables**:  
  - Rolling únicamente 3m (añadir 1/6/12m y *expanding windows*).  
  - Priors de segmento con más granularidad y **tendencias**.  
  - Posibles **interacciones** (valor × recencia, tipo × región).  
- **Calidad/Disponibilidad de datos**: outliers o huecos por segmento/país pueden afectar estabilidad.

---

## 12) Hoja de ruta (mejoras)

**Modelado**
- Probar **LightGBM/XGBoost** (o árboles Spark) para **no linealidad** e **interacciones**.  
- **Calibración** de probabilidades (Platt/Isotónica) para decisiones basadas en umbral.  
- **Ensamblados / Stacking**: usar LR como última capa explicable.

**Features**
- Más ventanas temporales (1/6/12m), múltiples lags, estacionalidad.  
- Señales de **recencia** enriquecidas y **frecuencia**.  
- Interacciones y **ratios** compuestos.

**Validación / Producción**
- **Monitoreo de drift** (PSI/KS mensuales) y **alertas**.  
- Backtesting **mensual** o *rolling windows*.  
- **MLOps**: versionado de artefactos, *feature store*, CI/CD para reentrenos.

---

## 13) Mapa del repositorio

```
next-digital-order-eb2b/
├─ dataset/                         # datos parquet/csv (ajusta ruta en notebooks)
├─ nootbooks/
│  ├─ 01_EDA.ipynb                  # exploración y sanidad de datos
│  ├─ 02_Modelado.ipynb             # LR + desbalance + backtesting + PR@k
│  └─ 02_Modelado_explicado.ipynb   # versión con celdas "Qué/Por qué/Cómo" y mini-teoría
├─ requirements.txt                 # dependencias
└─ README.md                        # este documento
```

---

## 14) Guía breve para presentar (5–7 minutos)

1) **Problema y métrica de decisión (1 min)**  
   - “Queremos priorizar clientes con alta prob. de próximo pedido **DIGITAL**.”  
   - Métricas de negocio: **PR@k** y **Lift@k** (impacto en campañas).

2) **Cómo evitamos fuga y cómo hacemos features (2 min)**  
   - Etiquetado por **cliente-mes** con **siguiente** pedido.  
   - Features: actividad/valor, tendencia, ciclo de vida y priors de segmento.

3) **Modelo y validación temporal (2 min)**  
   - LR con `weightCol`, backtesting en cortes de tiempo.  
   - Mostrar tabla de métricas y resaltar **Lift ≈ 1.55×**.

4) **Hallazgos y next steps (1–2 min)**  
   - Qué funciona (señales de valor/tendencia), qué mejorar (no linealidad, más ventanas, calibración).  
   - Cómo usar top-k en campañas y estimar ROI.

---

## 15) FAQ (rápidas)

- **¿Por qué LR y no XGBoost?**  
  Empezamos por LR por **rapidez/estabilidad/explicabilidad**. El roadmap contempla **árboles graduales** si hay presupuesto de complejidad.

- **¿Por qué AUC PR y PR@k?**  
  En desbalance, son **más fieles** que accuracy/AUC ROC para medir impacto operativo.

- **¿Cada cuánto reentrenar?**  
  Sugerencia: **mensual** o cuando se detecte **drift** en AUC PR/PR@k.

---

**Contacto:** edujerezwilson@gmail.com
