# Next Digital Order eB2B  

## 📌 Contexto  
La compañía busca aumentar el uso de pedidos digitales en la plataforma **eB2B**. Actualmente, muchos clientes siguen comprando por teléfono o a través de vendedores. El reto consiste en **identificar qué clientes tienen mayor probabilidad de que su próximo pedido sea digital**, para así priorizar esfuerzos comerciales y de comunicación.  

## 🎯 Objetivo  
A partir del dataset transaccional entregado, se desarrolló un flujo de trabajo para:  
1. **Explorar y analizar** los datos.  
2. **Construir un modelo** que estime la probabilidad de que el próximo pedido de un cliente sea digital.  
3. **Evaluar y comunicar resultados** con métricas relevantes para negocio y marketing.  

## 📂 Dataset  
Variables disponibles:  
- `cliente_id`  
- `pais_cd`  
- `fecha_pedido_dt`  
- `canal_pedido_cd` (DIGITAL, VENDEDOR, TELEFONO)  
- `facturacion_usd_val`  
- `materiales_distintos_val`  
- `dias_desde_ultimo_pedido`  
- `frecuencia_visitas_cd`  

## 🔎 Enfoque Metodológico  
1. **EDA (Exploratory Data Analysis)**: análisis de distribución de canales, frecuencia de compra, patrones por país y temporalidad.  
2. **Construcción de labels**: definición del target como *“canal del próximo pedido”* (evitando fuga de información).  
3. **Feature engineering**:  
   - Variables de recencia, frecuencia y monto.  
   - Ventanas temporales (7/30/90 días).  
   - Historial de digitalización del cliente.  
4. **Modelado**:  
   - Modelos base: Regresión Logística, Gradient Boosted Trees.  
   - Estrategias de balanceo de clases (ponderaciones y sampling).  
   - Búsqueda de hiperparámetros con validación temporal.  
5. **Evaluación**:  
   - Métricas: AUC-ROC, **AUC-PR**, F1, Recall@k (k=5, 10, 50).  
   - Curvas de calibración y análisis de umbrales óptimos.  
6. **Interpretabilidad**:  
   - Importancia de variables (Permutation / SHAP).  
   - Insights de negocio sobre factores que impulsan digitalización.  

## 📊 Principales Hallazgos (placeholder)  
- El **canal digital** muestra fuerte dependencia con la **recencia de pedidos digitales previos**.  
- Clientes con **≥2 pedidos digitales en los últimos 90 días** tienen significativamente mayor probabilidad de repetir digital.  
- Las métricas base muestran un **AUC-PR de ~0.41** y un F1 de ~0.48 (se trabaja en mejorar balanceo y calibración).  

## ⚠️ Limitaciones  
- Dataset limitado a variables transaccionales; no incluye datos de campañas ni variables sociodemográficas.  
- Horizonte temporal relativamente corto: dificulta capturar estacionalidad completa.  
- El modelo requiere pruebas adicionales de robustez por país.  

## 🚀 Próximas Mejoras  
- Implementar validación cruzada temporal.  
- Optimizar selección de umbrales para Recall@k según capacidad comercial.  
- Desplegar un **pipeline reproducible** con `src/` + `Makefile`.  
- Añadir un **demo CLI** o notebook interactivo para generar listas top-k de clientes a activar.  

## ⚙️ Reproducibilidad  
```bash
# Crear entorno
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Ejecutar notebooks
jupyter lab

# (Próximamente) ejecutar pipeline
make prep
make train
make eval
```
