# Predicción del **Próximo Pedido DIGITAL** (eB2B)

**Autor:** Wilson Eduardo Jerez Hernández  
**Fecha:** 2025-09-16

---

## 🧾 Resumen rápido (1 minuto)

- **Problema:** muchos clientes aún no usan el canal **DIGITAL** (eB2B).  
- **Objetivo:** estimar la **probabilidad** de que el **próximo pedido** de un cliente sea **digital** para **priorizar campañas**.  
- **Cómo:** trabajamos a nivel **cliente–mes**, construimos **señales** (actividad, tendencia y contexto) y entrenamos una **Regresión Logística** (*logistic regression*) que da una **probabilidad** por cliente.  
- **Resultado clave:** si llamamos solo al **top‑10%** con mayor probabilidad, la campaña es ~**1.55×** más efectiva que al azar (**Lift** (*lift*)).

---

## 0) Requisitos de la prueba técnica (y dónde se cumplen)

- **Explorar y analizar los datos** → `nootbooks/01_EDA.ipynb` (sanidad y entendimiento).  
- **Enfoque para estimar la probabilidad del próximo pedido digital** → `nootbooks/02_Modelado.ipynb` (features temporales, modelo, desbalance).  
- **Evaluar resultados y comunicar hallazgos** → sección **Backtesting** (más abajo) y este **README** (resumen claro).  
- **Entregables en GitHub**: código/notebooks + README con **enfoque**, **hallazgos** y **mejoras** (todo en este repo).

> Nota: el README usa lenguaje **simple**; los tecnicismos y anglicismos van entre paréntesis para ir aprendiéndolos poco a poco.

---

## 1) Problema y objetivo

La compañía quiere **acelerar la adopción** del canal **digital**.  
Con el histórico de pedidos por cliente, construimos un modelo que **ordena** a los clientes según la **probabilidad** de que **su próximo pedido** sea **digital**.  

👉 Así el equipo comercial **empieza por los de mayor probabilidad** y usa mejor el **presupuesto**.

---

## 2) Datos y preparación (qué / cómo)

**Nivel de análisis:** `cliente_id × mes` (cliente–mes).  

**Campos típicos (ejemplos):**  
`cliente_id, pais_cd, fecha_pedido_dt, canal_pedido_cd (DIGITAL, VENDEDOR, TELEFONO), facturacion_usd_val, materiales_distintos_val, cajas_fisicas, frecuencia_visitas_cd, ...`

**Normalización temporal (cómo):**
- `month_first = trunc(fecha_pedido_dt, 'month')`  
- `ym = date_format(month_first, 'yyyy-MM')`  
- `is_digital = 1 si canal == 'DIGITAL' (si no, 0)`

**¿Por qué cliente–mes?**  
- Permite construir **señales temporales** (recencia, promedios móviles (*rolling*), etc.).  
- Evita mezclar observaciones con diferentes escalas de tiempo.  
- Alinea la **ventana de decisión** con la ejecución real de campañas (mensual/quincenal).

---

## 3) Etiqueta sin **fuga de información** (*data leakage*)

- Por cada cliente–mes tomamos el **último pedido** del mes.  
- La **etiqueta** (`label`) es **1** si el **siguiente pedido** del cliente fue **digital**; si no, **0**.  
- Usamos solo **información pasada** del cliente para construir las features (nunca el futuro).

👉 Así respondemos *“¿el próximo pedido será digital?”* sin hacer **trampa**.

---

## 4) Features (señales usadas)

1. **Actividad y valor** → `n_orders`, `sum_fact`, `avg_fact`, `sum_cajas`, `avg_cajas`, `avg_mat_dist`.  
2. **Comportamiento digital** → `digital_ratio` (mes), `digital_ratio_3m` (promedio 3 meses), `lag1_digital_ratio` (mes anterior), `growth_digital_ratio` (cambio).  
3. **Ciclo de vida** → `months_since_first` (meses desde el primer pedido).  
4. **Contexto del segmento (priors)** → `% digital del mes anterior` en su **región** y en su **tipo de cliente**.  
5. **Categóricas** (baja cardinalidad) → `pais_cd`, `tipo_cliente_cd`, `madurez_digital_cd`, `frecuencia_visitas_cd` (con **OHE** (*one‑hot encoding*)).

👉 Mezclamos **historial propio + tendencias + contexto** → mejor **ranking** de clientes.

---

## 5) Modelo y desbalance

- **Modelo:** **Regresión Logística** (*logistic regression*).  
- **Por qué:** **rápida**, **estable** y **explicable** (coeficientes).  
- **Desbalance:** hay más `0` (no digital) que `1` (digital).  
- **Estrategias probadas:**  
  1) **Pesos por clase** (`weightCol`).  
  2) **Oversampling** (*oversampling*): duplicar algunos `1`.  
  3) **Undersampling** (*undersampling*): reducir algunos `0`.  

> **Tuning**: grid simple (`regParam`, `elasticNetParam`) y selección por **AUC‑PR** (*area under precision‑recall*), más informativa con desbalance.

---

## 6) Validación temporal (*backtesting*)

Para simular cómo rendiría el modelo “en el pasado”:  
- Entrenamos con **meses anteriores** a cada corte.  
- Evaluamos en **meses siguientes** al corte.  

**Cortes usados:** `2023‑08`, `2023‑10`, `2023‑12`, `2024‑01`.

---

## 7) Métricas y resultados

### Mini‑guía
- **PR@k (precision at k)**: en el **top‑k%** del ranking, ¿qué % realmente fue 1?  
- **Recall@k (recall at k)**: de **todos** los 1, ¿qué % quedó dentro del **top‑k%**?  
- **Lift@k** = **PR@k / Tasa base** → cuántas veces mejor que **al azar**.  
- **F1** (en umbral óptimo): equilibrio entre precisión y cobertura (solo si clasificas 1/0).

### Backtesting (estrategia con **pesos por clase**)
| Corte   | AUC ROC | AUC PR | Umbral F1 | F1@Umbral | PR@5% | Lift@5% | PR@10% | Lift@10% |
|---------|--------:|-------:|----------:|----------:|------:|--------:|-------:|---------:|
| 2023-08 | 0.6233  | 0.4997 | 0.00      | 0.5422    | 0.5796 | 1.5582 | 0.5790 | 1.5566 |
| 2023-10 | 0.6196  | 0.4735 | 0.38      | 0.5236    | 0.5492 | 1.5578 | 0.5478 | 1.5539 |
| 2023-12 | 0.6146  | 0.4375 | 0.38      | 0.4993    | 0.5100 | 1.5651 | 0.5067 | 1.5548 |
| 2024-01 | 0.6119  | 0.4135 | 0.38      | 0.4823    | 0.4829 | 1.5697 | 0.4791 | 1.5571 |

**Lecturas clave (en simple):**
- **Valor de negocio** → en top‑10% el **Lift ≈ 1.55×**: ~**55%** más efectivo que al azar.  
- **Precisión top‑10%** → ~**48–58%** según el corte: casi **la mitad o más** sí compra digital.  
- **Estabilidad** → cifras consistentes entre cortes; ojo con leve caída de **AUC‑PR** (posible **drift** (*cambio de comportamiento*)).

---

## 8) Conclusiones

1) El **ranking** por probabilidad **sirve** para campañas con presupuesto limitado.  
2) Las **tendencias digitales** y el **contexto de segmento** mejoran la prioridad de a quién contactar.  
3) Optimizar por **AUC‑PR** y reportar **PR@k/Lift@k** es más útil que **accuracy** en desbalance.  

---

## 9) Limitaciones y mejoras

- **Modelo lineal** → probar modelos de **árboles** (p. ej., **XGBoost** (*xgboost*)) para no linealidades.  
- **Señales temporales** → añadir ventanas **1/6/12m** y estacionalidad.  
- **Calibración** de probabilidades (Platt/Isotónica) si vas a usar **umbrales** para clasificar.  
- **Monitoreo** → revisar **drift** mensual (PSI/KS) y programar **reentrenos**.

---

## 10) Cómo ejecutar (rápido y claro)

**Requisitos**  
- Python 3.10+  
- PySpark (probado localmente)  
- Dependencias en `requirements.txt`

```bash
# 1) Instala dependencias
pip install -r requirements.txt

# 2) (Opcional) Ajusta rutas de datos en los notebooks
#    Datos esperados en: dataset/

# 3) Ejecuta los notebooks
#   - Exploración: nootbooks/01_EDA.ipynb
#   - Modelado  : nootbooks/02_Modelado.ipynb
```

**Salida esperada**
- Probabilidades por cliente (`p1`).  
- Ranking y **top‑k** para campaña.  
- Métricas de **PR@k**, **Lift@k** y **F1** (umbral óptimo).

---

## 11) Mapa del repositorio

```
next-digital-order-eb2b/
├─ dataset/                 # datos (parquet/csv) – ajusta ruta en notebooks si cambia
├─ nootbooks/               # 01_EDA.ipynb, 02_Modelado.ipynb
├─ reports/                 # (opcional) figuras/tablas para presentar
├─ requirements.txt
└─ README.md                # este documento
```

---

## 12) Guion de presentación (5 minutos)

1) **Qué resolvemos (30s):** “Priorizar clientes con alta probabilidad de próximo pedido **digital**.”  
2) **Cómo lo evitamos (fuga) y cómo armamos datos (1m):** cliente–mes, etiqueta con **siguiente pedido**.  
3) **Señales y modelo (1.5m):** actividad, tendencia, contexto + **Regresión Logística**.  
4) **Resultados (1m):** **Lift ~1.55×** en top‑10%; tabla de backtesting.  
5) **Cómo usarlo (30s):** ordenar por `p1`, elegir top‑k según presupuesto, medir **PR@k**/**Lift@k**.  
6) **Mejoras (1m):** árboles (xgboost), más ventanas, calibración y reentrenos.

---

## 13) Glosario rápido

- **Top‑k%**: el **porcentaje superior** de clientes con mayor probabilidad.  
- **Precisión (precision)**: de los que seleccioné, **qué %** fueron realmente 1.  
- **Cobertura (recall)**: de **todos** los 1, **qué %** seleccioné.  
- **Lift**: cuántas veces mejor que al azar.  
- **Fuga de información (data leakage)**: usar datos del **futuro** para decidir **hoy** (trampa).  
- **Backtesting**: simular el rendimiento **en el pasado**.  
- **Desbalance**: cuando hay **muchos más 0** que **1**.  
- **Regresión Logística (logistic regression)**: modelo que da **probabilidades**.

---

**Contacto:** edujerezwilson@gmail.com
