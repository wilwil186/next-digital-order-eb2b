# Predicción del **Próximo Pedido DIGITAL** (eB2B) — Guion para presentar (versión simple)

**Autor:** Wilson Eduardo Jerez Hernández  
**Fecha:** 2025-09-16

---

## 🧾 Resumen en 1 minuto 
- **Qué queremos:** ordenar los clientes por **probabilidad** de que su **próximo pedido** sea **DIGITAL**.
- **Para qué:** priorizar a quién contactar primero en una **campaña**, usando mejor el **presupuesto**.
- **Cómo:** análisis **cliente–mes**, señales simples (actividad, tendencia y contexto) y un modelo de **Regresión Logística** (*logistic regression*) que da una **probabilidad** por cliente.
- **Resultado clave:** si contactamos solo el **top-10%** de clientes (los de probabilidad más alta), la campaña es ~**1.55×** más efectiva que al azar.  
  > Abajo explico qué significa ese “**Lift ≈ 1.55×**” con palabras sencillas.

---

## 1) Problema y objetivo 
- Hoy algunos clientes compran **digital** y otros por **teléfono** o **vendedor**.  
- Queremos **predecir** quién es **más probable** que **pida digital la próxima vez**.  
- Con esa lista ordenada, **empezamos** por los de **mayor probabilidad** y así **mejoramos la campaña**.

> Regla de oro: **usamos solo el pasado** para decidir hoy (si miráramos el futuro sería “fuga de información” (*data leakage*), o sea, trampa).

---

## 2) Datos y forma de trabajo

- Trabajamos a nivel **cliente–mes** (todos comparables en la misma unidad de tiempo).
- Para cada cliente y mes calculamos:
  - **Actividad y valor**: nº de pedidos, facturación, cajas, etc.  
  - **Comportamiento digital**: % digital del mes y su tendencia reciente.  
  - **Tiempo con la empresa**: meses desde su primer pedido (**ciclo de vida**).  
  - **Contexto**: cómo le fue a su **región** y a su **tipo de cliente** el **mes anterior** (porcentaje digital en su grupo).

---

## 3) Cómo armamos la **etiqueta** (la “respuesta”)

- Por cada **cliente–mes** tomamos el **último pedido** de ese mes.  
- Miramos **el siguiente** pedido del cliente:
  - Si fue **digital** → **1**  
  - Si **no** fue digital → **0**  
- Así contestamos a “**¿El próximo pedido será digital?**”, sin hacer trampa.

---

## 4) El modelo (explicado fácil)

- **Regresión Logística** (*logistic regression*):  
  - No dice “sí/no” directamente; **da una probabilidad** de 0 a 1.  
  - Con esa probabilidad **ordenamos** clientes (del mayor al menor).
- ¿Por qué este modelo?  
  - Es **rápido**, **estable** y **fácil de explicar** (sus números muestran qué señales ayudan o restan).

---

## 5) Desbalance (pocos 1’s, muchos 0’s)

- Normalmente hay **menos** casos “digital” (1) que “no digital” (0).  
- Para que el modelo **escuche** a la clase 1:  
  - Damos **más peso** a los 1 (opción `weightCol`).  
  - También se puede **duplicar** algunos 1 (**oversampling** (*oversampling*)) o **reducir** algunos 0 (**undersampling** (*undersampling*)).

---

## 6) Validación en el tiempo (mini simulación del pasado)

- Para cada **corte** en el tiempo:  
  - Entrenamos con **meses anteriores** y evaluamos en **meses siguientes**.  
- Esto se llama **backtesting** (*backtesting*) y nos dice si la idea **resiste el paso del tiempo**.

---

## 7) Métricas que sí sirven cuando hay campaña

Cuando el objetivo es **contactar a pocos** (por presupuesto), importa **arriba** del ranking:

- **Top-k%**: el **porcentaje superior** de clientes con mayor probabilidad (ej.: top-10%).  
- **Precisión en top-k (PR@k)** (*precision at k*):  
  De ese top-k, **qué %** realmente fueron 1.  
- **Cobertura en top-k (Recall@k)** (*recall at k*):  
  De **todos** los 1, **qué %** cayó dentro del top-k.
- **Lift@k** = **PR@k / Tasa base**.  
  - **Tasa base**: % de 1’s en **toda** la base (si seleccionaras al azar).

---

## 8) ¿Qué significa **Lift ≈ 1.55×**? (versión para todo público)

- **Idea corta:** “**Nuestra selección del top-k es 1.55 veces mejor que al azar**.”  
- **Traducción de negocio:** “Si en vez de llamar al azar usamos el ranking del modelo, **aprovechamos ~55% mejor el esfuerzo** en ese grupo top-k.”

**Ejemplo con números redondos**  
- En toda la base, **30%** de los clientes terminan comprando digital (tasa base = 0.30).  
- En el **top-10%** elegido por el modelo, **45%** compra digital (PR@10% = 0.45).  
- **Lift@10% = 0.45 / 0.30 = 1.50** → **1.5×** mejor que al azar.  
  Si en tus datos el promedio fue ~**1.55×**, es lo mismo pero un poco mejor.

**Cómo decirlo en 10 segundos (para tu pitch):**  
> “Cuando contactamos solo al 10% más prometedor según el modelo, **convertimos ~55% mejor** que si llamáramos al azar. Eso es lo que significa **Lift ~1.55×**.”

**Mini visual (texto):**
```
Efectividad (azar)   : ██████████  (base)
Efectividad (modelo) : ████████████████ (~1.55×)
```

---

## 9) Resultados (lectura humana)

- **Lift** en el **top-10%** ≈ **1.55×** → **campaña más eficiente** en el grupo que realmente contactas.  
- **Precisión** en ese top-10% ≈ **48%–58%** (según el mes) → casi **la mitad o más** de los contactados **sí** hacen pedido digital.  
- **Estabilidad**: el desempeño se mantiene razonable en diferentes cortes de tiempo; si baja un poco, es normal (los hábitos cambian).

---

## 10) Cómo usarlo (paso a paso en la operación)

1. **Saca la probabilidad** por cliente (columna `p1`).  
2. **Ordena** de mayor a menor.  
3. Define tu **presupuesto** (ej.: top-10%).  
4. **Contacta** ese grupo top-k primero.  
5. Mide **PR@k** y **Lift@k** para ver el **impacto real** de la campaña.

---

## 11) Limitaciones y mejoras claras

- Modelo **lineal**: funciona bien y es claro, pero no capta relaciones muy complejas.  
  - **Mejora**: probar modelos de **árboles** (como **XGBoost** (*xgboost*)).
- **Cambios en el tiempo**: el comportamiento puede variar (**drift**).  
  - **Mejora**: **reentrenar** periódicamente y **monitorear** las métricas.
- **Señales**: hoy usamos ventanas de **3 meses**.  
  - **Mejora**: sumar ventanas de **1/6/12 meses**, estacionalidad e **interacciones**.

---

## 12) Para ejecutarlo tú (rápido)

1) Instala dependencias:
```bash
pip install -r requirements.txt
```
2) Coloca los datos en `dataset/` (o ajusta la ruta en el notebook).  
3) Abre el notebook **02_Modelado** y ejecútalo.  
4) Exporta tu **top-k** para campaña y mide **PR@k** y **Lift@k**.

---

## 13) Glosario express

- **Top-k%:** el **porcentaje superior** de clientes con más probabilidad.  
- **Precisión (precision):** de los que seleccioné, **qué %** fueron realmente 1.  
- **Cobertura (recall):** de **todos** los 1, **qué %** seleccioné.  
- **Lift:** cuántas veces mejor que al azar.  
- **Fuga de información (data leakage):** usar datos del **futuro** para decidir **hoy** (trampa).  
- **Backtesting:** simular el rendimiento **en el pasado**.  
- **Desbalance:** cuando hay **muchos más 0** que **1**.  
- **Regresión Logística (logistic regression):** modelo que da **probabilidades**.

---

**Contacto:** edujerezwilson@gmail.com
