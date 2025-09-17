# Evaluación de Resultados y Hallazgos

## 🔎 Exploratory Data Analysis (EDA)
1. **Distribución de canales de pedido**
   - El canal **Vendedor** es el más frecuente, seguido por **Teléfono**.
   - El canal **Digital** aún representa un porcentaje menor, pero con tendencia de crecimiento.

2. **Patrones temporales**
   - Se observa estacionalidad ligera en algunos meses, con picos de facturación en periodos específicos.
   - Los clientes que ya han usado el canal digital en el pasado presentan mayor probabilidad de repetirlo en periodos cortos de tiempo.

3. **Variables relevantes**
   - **`dias_desde_ultimo_pedido`**: clientes con menor tiempo entre pedidos tienden a usar más el canal digital.
   - **`facturacion_usd_val`**: pedidos de menor facturación tienen mayor propensión a ser digitales.
   - **`materiales_distintos_val`**: pedidos con menor variedad de materiales suelen ser digitales.
   - **`frecuencia_visitas_cd`**: mayor frecuencia de visitas se asocia a pedidos por vendedor.

## 🤖 Modelado (Inferencia)
1. **Enfoque utilizado**
   - Target: *“canal del próximo pedido”* → 1 si es **DIGITAL**, 0 en otro caso.
   - Modelos probados: **Regresión Logística** y **Gradient Boosted Trees**.
   - Estrategias de balanceo de clases: `class_weight`, oversampling.

2. **Métricas obtenidas**
   - **AUC ROC**: ~0.62
   - **AUC PR**: ~0.41
   - **Accuracy**: ~0.58
   - **Precision**: ~0.39
   - **Recall**: ~0.63
   - **F1**: ~0.48

   > El recall es relativamente alto, lo que permite identificar gran parte de los clientes que efectivamente harán pedidos digitales, aunque la precisión todavía es baja.

3. **Hallazgos clave del modelo**
   - Los **pedidos recientes digitales** son la señal más fuerte para predecir digitalización futura.
   - Variables de **recencia y frecuencia** aportan más que las de monto o diversidad de materiales.
   - Existen diferencias de comportamiento por país que sugieren entrenar modelos locales o agregar interacciones país × canal.

## 📌 Conclusiones
- El modelo actual tiene **buen potencial como herramienta de priorización**: permite identificar un subconjunto de clientes con alta probabilidad de digitalizarse.
- Las métricas, especialmente el **AUC-PR y Recall@k**, son más útiles para campañas que la exactitud global.
- Recomendación: optimizar el umbral para campañas de marketing y explorar segmentación por país.

## 🚀 Próximos pasos sugeridos
- Implementar validación temporal más robusta.
- Optimizar umbrales de decisión según la capacidad comercial (Recall@k).
- Incorporar más fuentes de datos (campañas, variables de cliente).
- Desplegar pipeline reproducible con outputs (predicciones y métricas) en `reports/`.
