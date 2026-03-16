# 🛡️ MLOps para Detección de Fraude en Tiempo Real
### PayCol Fintech — Arquitectura de Procesamiento Distribuido

> **Proyecto**: Diseño de sistema MLOps de detección de fraude end-to-end  
> **Empresa**: PayCol — Fintech colombiana fundada en 2019  
> **Stack principal**: Azure + Apache Kafka + Spark + Databricks + MLflow + XGBoost/LightGBM  
> **Última actualización**: Marzo 2026

---

## 📋 Tabla de Contenidos

1. [Contexto y Problema de Negocio](#1-contexto-y-problema-de-negocio)
2. [Requerimientos Técnicos](#2-requerimientos-técnicos)
3. [Arquitectura MLOps Propuesta](#3-arquitectura-mlops-propuesta)
4. [Componentes del Pipeline](#4-componentes-del-pipeline)
   - [Fuentes de Datos](#41-fuentes-de-datos)
   - [Almacenamiento y Gobernanza](#42-almacenamiento-y-gobernanza)
   - [Feature Engineering Pipeline](#43-feature-engineering-pipeline)
   - [Model Iterations](#44-model-iterations--experimentación)
   - [Model Deployment](#45-model-deployment--producción)
   - [Performance Monitoring](#46-performance-monitoring)
5. [Clasificación de Componentes](#5-clasificación-de-componentes--iaas--paas--saas)
6. [Modelo de Procesamiento](#6-modelo-de-procesamiento)
7. [Escalabilidad y Resiliencia](#7-escalabilidad-y-resiliencia)
8. [Estimación de Costos](#8-estimación-de-costos)
9. [Documentación del Proyecto](#9-documentación-del-proyecto)
10. [Conclusiones](#10-conclusiones)

---

## 1. Contexto y Problema de Negocio

**PayCol** es una fintech colombiana con **4 millones de usuarios activos** que procesa un promedio de **2 millones de transacciones diarias**, con picos de hasta **8.000 transacciones por segundo (TPS)** en eventos como Black Friday o fechas de pago de nómina.

### El Problema

| Indicador | Situación Actual | Objetivo |
|---|---|---|
| Pérdidas anuales por fraude | **COP $4.200 millones** | Reducir ≥ 70% |
| Tasa de detección | **62%** | **≥ 90%** |
| Falsos positivos | **18%** | **≤ 3%** |
| Latencia de evaluación | Sin restricción | **< 200ms (p99)** |
| Sistema actual | Reglas estáticas manuales | ML adaptativo en tiempo real |

Los sistemas de reglas estáticas son **incapaces de adaptarse** al comportamiento cambiante del fraude. Los defraudadores aprenden las reglas y las evaden, mientras que el sistema actual bloquea el 18% de transacciones legítimas (~720K usuarios afectados/año) y deja pasar el 38% del fraude real.

### La Solución

Un sistema **MLOps de detección de fraude en tiempo real** que:
- ✅ Evalúa cada transacción en **< 200ms** mediante inferencia distribuida en streaming
- ✅ Alcanza **≥ 90% de detección** con **≤ 3% de falsos positivos**
- ✅ Se **reentrena automáticamente cada hora** para adaptarse al fraude evolutivo
- ✅ Cumple **Habeas Data, SARLAFT** y reportes a la Superfinanciera
- ✅ Garantiza **99.99% de disponibilidad** (≤ 52 min downtime/año)

> **ROI estimado**: ~950% en el primer año · Payback < 1.2 meses

---

## 2. Requerimientos Técnicos

### Las 3 V's del Big Data para PayCol

#### Volumen
| Parámetro | Valor |
|---|---|
| Transacciones diarias | **2 millones** (~23 TPS promedio) |
| TPS en pico (Black Friday) | **8.000 TPS** (350× el promedio) |
| Throughput streaming pico | **~100 MB/s** |
| Datos históricos etiquetados | **~3 TB** |
| Retención transacciones | **5 años** |
| Retención features ML | **90 días** |

#### Velocidad
| Restricción | Valor |
|---|---|
| Latencia end-to-end (p99) | **< 200ms** |
| Disponibilidad | **99.99%** (≤ 52 min downtime/año) |
| Frecuencia de reentrenamiento | **Cada hora** |
| Kafka (ingesta) | < 10ms |
| Feature retrieval (Redis) | < 15ms |
| Inferencia ONNX | < 30ms |

**Presupuesto de latencia desglosado:**
```
200ms totales:
├── Kafka ingesta:           ~10ms
├── Feature Store (Redis):   ~15ms
├── Spark Streaming:         ~25ms
├── Inferencia ONNX:         ~30ms
├── Post-processing:         ~10ms
├── Escritura resultado:     ~20ms
└── Overhead red:            ~20ms
                    Total: ~130ms (buffer: 70ms)
```

#### Variedad
| Tipo de dato | Fuente | Procesamiento |
|---|---|---|
| Transacciones | App PayCol | Streaming |
| Historial usuario | PostgreSQL | Batch |
| Geolocalización + dispositivo | App móvil | Streaming |
| Listas negras | Superfinanciera | Batch diario |
| Etiquetas de fraude | Equipo investigación | Batch |

**Restricciones regulatorias (Variedad de cumplimiento)**:
- 🏛️ **Habeas Data** (Ley 1581/2012): Cifrado, anonimización y auditoría de acceso
- 🏛️ **SARLAFT**: Trazabilidad completa de decisiones de bloqueo
- 🏛️ **Superfinanciera**: Reportes periódicos de métricas de fraude

---

## 3. Arquitectura MLOps Propuesta

### Diagrama del Pipeline MLOps (basado en `MLOPsv2.png`)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA MLOPS — PAYCOL FRAUDE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 6: MONITOREO Y OBSERVABILIDAD                                         │
│  Evidently AI | Azure Monitor | Grafana | PagerDuty                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 5: REENTRENAMIENTO AUTOMATIZADO (cada hora)                           │
│  Azure Databricks | MLflow | PyCaret | XGBoost/LightGBM                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 4: SERVING EN TIEMPO REAL (< 200ms)                                   │
│  AKS (Kubernetes) | FastAPI | ONNX Runtime | Redis                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 3: STREAMING E INGENIERÍA DE FEATURES                                 │
│  Apache Kafka (8.000 TPS) | Spark Structured Streaming | Feature Store      │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 2: ALMACENAMIENTO Y GOBERNANZA                                         │
│  Azure Data Lake Storage Gen2 | Azure Synapse Analytics | Key Vault         │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAPA 1: FUENTES DE DATOS                                                    │
│  App PayCol | PostgreSQL | APIs externas | Superfinanciera                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ████ SEGURIDAD TRANSVERSAL: Azure AD | TLS 1.3 | AES-256 | RBAC ████      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flujo de una Transacción (< 200ms)

```
📱 App PayCol
    ↓ JSON/HTTPS [0ms]
⚡ Apache Kafka (32 particiones)
    ↓ stream [10ms]
🌊 Spark Structured Streaming
    ↓ + features de Redis [35ms]
🎯 Modelo ONNX (XGBoost+LightGBM)
    ↓ score [75ms]
📊 Decisión: APROBAR / BLOQUEAR [130ms]
    ↓
💾 Log en ADLS + Redis async [200ms]
```

### Flujo de Reentrenamiento (cada hora)

```
📦 ADLS Gen2 (últimas 4h de datos)
    ↓
🔥 Azure Databricks (Spark Batch + PyCaret)
    ↓ XGBoost + LightGBM en paralelo
📊 MLflow (tracking + Model Registry)
    ↓ si métricas > threshold
🚀 Deploy automático → ONNX → AKS
    ↓
📈 Evidently AI (monitoreo drift)
    ↓ si degradación
🔁 Loop: volver al inicio
```

---

## 4. Componentes del Pipeline

### 4.1 Fuentes de Datos

| Fuente | Tipo | Datos | Ingesta |
|---|---|---|---|
| **App PayCol** | Transaccional | JSON con monto, comercio, GPS, device | Streaming via Kafka |
| **PostgreSQL** | Relacional | Historial usuario, perfil de comercios | Batch/CDC via Debezium |
| **APIs Superfinanciera** | Regulatoria | Listas negras, alertas SARLAFT | Batch diario |
| **Datasets públicos/privados** | Enriquecimiento | Datos demográficos, fraude conocido | Batch semanal |

### 4.2 Almacenamiento y Gobernanza

| Servicio | Rol | Justificación |
|---|---|---|
| **Azure Data Lake Storage Gen2** | Almacenamiento maestro (Bronze/Silver/Gold) | Integración nativa con Databricks, $0.018/GB |
| **Azure Synapse Analytics** | Consultas analíticas sobre historial 3TB | SQL On-demand, sin servidor que gestionar |
| **Azure Cache for Redis** | Feature Store online (< 1ms latencia) | In-memory, clave para el presupuesto de 200ms |
| **Azure Key Vault** | Gestión de secretos y certificados | HSM gestionado, FIPS 140-2 compliant |

**Arquitectura Medallion en ADLS Gen2:**
```
/bronze/  → Datos raw tal como llegan (JSON/Avro)
/silver/  → Datos limpios y validados (Parquet/Delta)
/gold/    → Features procesadas para ML (Delta Lake)
```

### 4.3 Feature Engineering Pipeline

El pipeline de ingeniería de características transforma datos brutos en **85 features de alta calidad** para el modelo de fraude:

#### Etapas

| Etapa | Descripción | Herramienta |
|---|---|---|
| **Data Cleaning** | Nulos, duplicados, outliers, tipos | pandas + PySpark |
| **Feature Extraction** | Ratios de monto, velocidad de gasto, hora del día | Spark Structured Streaming |
| **Aggregations** | Ventanas temporales: 1h, 24h, 7d, 30d por usuario/comercio | Spark Window Functions |
| **PCA** | Reducción de dimensionalidad para features correlacionadas | scikit-learn |
| **Feature Selection** | LASSO, RFE, mutual information | scikit-learn + Optuna |
| **Feature Store** | Redis (online) + ADLS Gold (offline) | Redis + Delta Lake |

**Features críticas para detección de fraude:**
- `tx_count_1h` — Nº de transacciones en la última hora
- `tx_amount_avg_7d` — Monto promedio últimos 7 días
- `location_anomaly_score` — Distancia geográfica vs. patrón habitual
- `device_change_flag` — ¿El dispositivo cambió recientemente?
- `time_since_last_tx_min` — Minutos desde la última transacción
- `login_failures_1h` — Intentos fallidos de login recientes

### 4.4 Model Iterations — Experimentación

El bloque de experimentación sigue el flujo del diagrama MLOps v2:

```
Data Validation → Data Preparation → Model Training → Model Evaluation → Model Validation
```

#### Experimentos Orquestados

| Paso | Herramienta | Descripción |
|---|---|---|
| **Data Validation** | Great Expectations | Schema, rangos, cardinalidad, drift vs. baseline |
| **Data Preparation** | PySpark + scikit-learn | Balanceo SMOTE, encoding, scaling, split estratificado |
| **Model Training** | PyCaret + XGBoost + LightGBM | Múltiples algoritmos en paralelo, k-fold CV |
| **Hyperparameter Tuning** | Optuna (Bayesian) | N trials en paralelo en Databricks |
| **Model Evaluation** | MLflow + Matplotlib | AUC-PR, F1, Recall, SHAP values |
| **Model Validation** | MLflow + pytest | Umbral mínimo: F1 ≥ 0.88, AUC-PR ≥ 0.90 |

#### Model Selection and Evaluation

```
┌──────────────────────────────────────────────────────────────┐
│  Model Selection and Evaluation                              │
│  ┌──────────────┐  ┌────────────────────────────────────┐   │
│  │ Comparación  │  │ Selección según requisitos          │   │
│  │ de modelos   │  │ (F1 ≥ 0.88, latencia < 15ms)       │   │
│  └──────────────┘  └────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Save model + params + artifacts → MLflow Registry   │   │
│  └──────────────────────────────────────────────────────┘   │
│  Herramientas: PyCaret | MLflow | Matplotlib | SHAP         │
└──────────────────────────────────────────────────────────────┘
         ↓ ¿Cumple criterios de calidad?
         YES → Model Deployment
         NO  → Volver a Model Iterations
```

**Métricas objetivo (clasificación desbalanceada — fraude < 2%):**
- ✅ **AUC-PR** ≥ 0.90 (métrica principal — más robusta que AUC-ROC para clases desbalanceadas)
- ✅ **Recall (Fraude)** ≥ 90% (minimizar fraudes no detectados)
- ✅ **Precision (Fraude)** tal que FPR ≤ 3% (minimizar bloqueos de usuarios legítimos)
- ✅ **Latencia p99** < 15ms (inferencia ONNX, dentro del presupuesto de 200ms)

**¿Por qué XGBoost + LightGBM Ensemble?**
| Modelo | AUC-PR | Latencia | Interpretabilidad |
|---|---|---|---|
| Regresión Logística | Media | < 1ms | Alta |
| Random Forest | Alta | ~20ms ⚠️ | Parcial |
| **XGBoost** | **Muy alta** | **< 10ms** ✅ | **SHAP** ✅ |
| **LightGBM** | **Muy alta** | **< 5ms** ✅ | **SHAP** ✅ |
| Redes Neuronales (DNN) | Máxima | ~100ms ❌ | Baja ❌ |
| **Ensemble XGB+LGBM** | **Máxima** | **< 15ms** ✅ | **SHAP** ✅ |

### 4.5 Model Deployment — Producción

#### Pipeline Automatizado de Producción

Cada hora, el Automated Pipeline repite el ciclo completo sin intervención humana:

| Etapa | Herramienta | Descripción |
|---|---|---|
| **Data Extraction** | ADLS Gen2 + Delta Lake | Lee últimas 4h de transacciones (~480K registros) |
| **Data Validation** | Great Expectations | Falla y alerta si hay anomalías de datos |
| **Data Preparation** | PySpark + Feature Store | Aplica transformaciones del Feature Store |
| **Model Training** | Databricks + XGBoost + LightGBM | Entrenamiento distribuido en 2 clusters paralelos |
| **Model Evaluation** | MLflow | Evalúa vs. métricas baseline en producción |
| **Model Validation** | MLflow + Gates | Solo promueve si supera umbrales |

#### Serialización y Export

```
Modelo entrenado
    ↓ Conversión ONNX (Open Neural Network Exchange)
Formato ONNX (cross-platform, ultra eficiente)
    ↓ Push al Model Registry (MLflow)
Azure Container Registry (Docker image actualizada)
    ↓ Rolling deployment
AKS: actualización sin downtime (rolling update)
```

**¿Por qué ONNX sobre pickle?**
- ONNX es **cross-language** (Python entrenamiento → C++ runtime)
- **2-5× más rápido** que pickle en inferencia
- Compatible con **ONNX Runtime** de Microsoft, optimizado para Azure

#### CD: Model Serving

```
📱 Petición de transacción
        ↓
⚖️ Azure Load Balancer
        ↓
☸️ AKS (3-50 pods FastAPI según carga)
        ↓
🚀 FastAPI (async, Pydantic validation)
        ↓
🎯 ONNX Runtime (modelo en memoria)
        ↓
📊 Decisión: {fraud_score: 0.94, decision: "BLOCK"}
        ↓
💾 Async log → ADLS Gen2 (auditoría SARLAFT)
```

### 4.6 Performance Monitoring

El monitoreo cierra el ciclo MLOps con retroalimentación continua:

```
┌──────────────────────────────────────────────────────────────┐
│   Performance Monitoring                                      │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Model monitoring: Check for drifts, data quality    │  │
│  │  and model degradation (Evidently AI, cada 15 min)   │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Herramientas: Evidently AI | Azure Monitor | Grafana        │
└──────────────────────────────────────────────────────────────┘
         ↓ Degradación detectada
    → Trigger automático de reentrenamiento
    → Alerta a equipo ML (PagerDuty / Teams)
```

| Tipo de Monitoreo | Qué detecta | Frecuencia | Umbral |
|---|---|---|---|
| **Data Drift** | Cambio en distribución de features (PSI, KS test) | Cada 15 min | PSI > 0.2 |
| **Data Quality** | Nulos, rangos inválidos, cardinalidad | Cada 1 min | Configurado por feature |
| **Model Degradation** | Concept drift, caída de AUC-PR | Cada hora | AUC-PR < 0.85 |
| **Latencia p99** | Degradación de rendimiento de serving | Continuo | > 180ms |

---

## 5. Clasificación de Componentes — IaaS / PaaS / SaaS

| Componente | Clasificación | Justificación |
|---|---|---|
| **Azure VMs (nodos AKS)** | **IaaS** | Usuario gestiona SO y runtime |
| **Apache Kafka** (self-managed en AKS) | **IaaS** | Equipo gestiona brokers, configuración |
| **Azure Kubernetes Service (AKS)** | **PaaS** | Azure gestiona el control plane |
| **Azure Data Lake Storage Gen2** | **PaaS** | Almacenamiento gestionado sin servidor |
| **Azure Databricks** | **PaaS** | Spark gestionado + MLflow integrado |
| **Azure Synapse Analytics** | **PaaS** | Warehouse analítico gestionado |
| **Azure Cache for Redis** | **PaaS** | Redis gestionado con HA multi-AZ |
| **Azure ML Pipelines** | **PaaS** | Orquestación ML como servicio |
| **Azure Key Vault** | **PaaS** | HSM gestionado por Azure |
| **Azure Monitor + Log Analytics** | **SaaS** | Monitoreo completamente gestionado |
| **Azure Active Directory** | **SaaS** | Identidad como servicio |
| **PagerDuty** | **SaaS** | Gestión de incidentes completamente gestionado |
| **GitHub / Azure DevOps** | **SaaS** | CI/CD como servicio |

> **Conclusión**: Arquitectura predominantemente **PaaS**, reduciendo la carga operacional del equipo para enfocarse en los modelos y las features de fraude.

---

## 6. Modelo de Procesamiento

### ¿Qué es Streaming, Batch y Paralelo en PayCol?

| Componente | Tipo | Frecuencia | Latencia objetivo |
|---|---|---|---|
| **Kafka Consumer (ingesta)** | ⚡ **Streaming** | Por evento | < 10ms |
| **FastAPI + ONNX (inferencia)** | ⚡ **Streaming** | Por petición | < 80ms |
| **Spark Streaming (features)** | 🔄 **Micro-batch** | Cada 1 segundo | < 50ms |
| **Redis materialización** | 🔄 **Micro-batch** | Cada 30 segundos | < 1s |
| **Databricks reentrenamiento** | 📦 **Batch** | Cada hora | ~20-30 min |
| **Synapse Analytics histórico** | 📦 **Batch** | Diario | ~2-4 horas |
| **Listas negras Superfinanciera** | 📦 **Batch** | Diario | ~30 min |
| **Feature engineering distribuido** | ⚙️ **Paralelo** | En cada batch | Escala con N workers |
| **Entrenamiento XGB+LGBM** | ⚙️ **Paralelo** | Cada hora | Escala en Databricks |
| **AKS pod scaling** | ⚙️ **Paralelo** | Por demanda (HPA) | < 60s spawn |

### ¿Por qué no todo en streaming?
La inferencia (decisión de fraude) **DEBE** ser streaming — el usuario espera la respuesta en < 200ms. El reentrenamiento **NO puede** ser streaming porque:
1. Necesita suficientes ejemplos de fraude recientes para ser estadísticamente significativo.
2. Los algoritmos de gradient boosting (XGBoost, LightGBM) no son online-learning nativos.
3. El presupuesto de 200ms es para inferencia, no para entrenamiento.

---

## 7. Escalabilidad y Resiliencia

### Escenario de Carga Baseline vs. Pico

| Métrica | Baseline Normal | Pico Black Friday | Factor |
|---|---|---|---|
| TPS | ~23 TPS | **8.000 TPS** | **350×** |
| Pods FastAPI | 3 pods | **50 pods** | 17× |
| Workers Spark | 4 workers | **20 workers** | 5× |
| Particiones Kafka | 8 | **32** | 4× |

### Estrategia de Auto-scaling

**KEDA (Kubernetes Event-Driven Autoscaler)** escala pods basado en el **lag de Kafka** — más inteligente que CPU porque el lag refleja la demanda real:

```yaml
spec:
  triggers:
  - type: kafka
    metadata:
      topic: transacciones.raw
      lagThreshold: "100"    # Si hay > 100 mensajes sin procesar, escala
  minReplicaCount: 3         # Alta disponibilidad mínima
  maxReplicaCount: 50        # Máximo en Black Friday
```

### Tolerancia a Fallos

| Escenario | Estrategia | RTO | RPO |
|---|---|---|---|
| Caída de 1 pod FastAPI | HPA mantiene ≥ 3 réplicas, Load Balancer redirige | < 1s | 0 |
| Caída de 1 broker Kafka | RF=3 → los otros 2 brokers tienen réplica completa | < 30s | 0 |
| Fallo del Feature Store (Redis) | **Fallback a reglas estáticas** | < 5s | 0 |
| Fallo del modelo ML | **Fallback a reglas estáticas** (Circuit Breaker) | < 5s | 0 |
| Caída de una Zona de Disponibilidad | Multi-AZ deployment en 3 AZ distintas | < 5 min | < 30s |
| Modelo corrompido en producción | MLflow rollback automático a versión anterior | < 2 min | 0 |

**Pattern Circuit Breaker** (clave para el 99.99% SLA):
```
Transacción → FastAPI → [Circuit Breaker] → Modelo ML (ONNX)
                              ↓ (si ML falla 5× en 10s)
                        [FALLBACK: Reglas estáticas]  ← degradación graceful
                              ↓ (cuando ML recupera)
                        [Cerrar Circuit, retornar a ML]
```

### Pre-calentamiento para Eventos de Alta Demanda

- **72h antes**: Pre-escalar Kafka a 32 particiones y AKS a 10 réplicas mínimas.
- **24h antes**: Test de carga con 120% del TPS esperado.
- **Durante**: Feature flags para activar/desactivar features dinámicamente.
- **Post-evento**: Scale-down programado para optimización de costos.

---

## 8. Estimación de Costos

> Costos en USD en Azure (East US 2 / Brazil South). Fuente: [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator).

| Componente | Configuración | Costo/Mes (baseline) |
|---|---|---|
| **Azure Databricks** | 2 clusters × 8 cores, spot 80%, 1h/hora | ~$580 |
| **AKS Compute** | 3-10 nodos Standard_D4s_v3 | ~$420 - $1.400 |
| **Azure Cache for Redis** | Premium P2 (13 GB, HA multi-AZ) | ~$270 |
| **ADLS Gen2** | 30 TB LRS + features 90 días | ~$540 |
| **Azure Monitor + Log Analytics** | 50 GB/día logs | ~$3.450 |
| **Azure Synapse Analytics** | Serverless SQL 500 GB/mes | ~$2.50 |
| **Networking + otros** | Egress, LB, Key Vault, DevOps | ~$220 |
| **TOTAL ESTIMADO** | | **~$5.500 - $7.500/mes** |

### Proyección Anual

| Escenario | Costo/Año |
|---|---|
| Baseline (sin picos) | **~$65.760 USD** |
| Con picos (10% meses pico) | **~$68.400 USD** |

### ROI del Proyecto

| Concepto | Valor |
|---|---|
| Pérdidas actuales por fraude/año | COP $4.200M ≈ **USD $1.050.000** |
| Reducción esperada (70% menos fraude) | **USD $735.000/año** |
| Costo anual infraestructura | **USD ~$68.000/año** |
| **ROI neto anual** | **~USD $665.000 (950% ROI)** |
| **Payback period** | **< 1.2 meses** |

### Optimizaciones de Costo

- 💰 **Spot instances** en Databricks: Ahorro 60-80% en entrenamiento
- 💰 **Reserved Instances** para nodos AKS baseline: Ahorro 40% (1 año)
- 💰 **ADLS Lifecycle**: Datos > 90 días a tier frío → 50% menos en almacenamiento antiguo
- 💰 **Kafka self-managed en AKS**: Se evita Confluent Cloud (~$2.000/mes adicionales)

---

## 9. Documentación del Proyecto

| Archivo | Descripción |
|---|---|
| [`MLOPsv2.png`](./MLOPsv2.png) | Diagrama visual de la arquitectura MLOps |
| [`MLOPS_Diagram.md`](./MLOPS_Diagram.md) | Análisis profundo de cada componente del diagrama MLOps |
| [`CasodeUso.md`](./CasodeUso.md) | Definición del caso de uso: PayCol fraud detection |
| [`MLOPS_CasodeUso.md`](./MLOPS_CasodeUso.md) | Solución técnica completa con los 10 puntos de la rúbrica |
| [`RubricadeEvaluación.md`](./RubricadeEvaluación.md) | Criterios de evaluación del proyecto (100 pts) |

### Criterios de Evaluación (100 puntos)

| Criterio | Puntos | Dónde se cubre |
|---|---|---|
| Arquitectura técnica | 25 pts | Sección 3 + `MLOPS_Diagram.md` |
| Justificación de componentes | 20 pts | Sección 4 + `MLOPS_CasodeUso.md` §4 |
| Clasificación IaaS/PaaS/SaaS | 20 pts | Sección 5 + `MLOPS_CasodeUso.md` §5 |
| Escalabilidad y resiliencia | 15 pts | Sección 7 + `MLOPS_CasodeUso.md` §7 |
| Estimación de costos | 10 pts | Sección 8 + `MLOPS_CasodeUso.md` §8 |
| Claridad y presentación | 10 pts | Este README + todos los documentos |

---

## 10. Conclusiones

### Decisiones Técnicas Clave

| Decisión | Elección | Alternativa Considerada | Razón de Elección |
|---|---|---|---|
| Message Broker | **Apache Kafka** | AWS Kinesis, Confluent | Throughput ilimitado, replay, open source |
| Procesamiento Streaming | **Spark Structured Streaming** | Apache Flink | Integración nativa con Databricks + MLlib |
| Almacenamiento | **ADLS Gen2** | AWS S3 | Ecosistema Azure coherente, $0.018/GB |
| Feature Store Online | **Redis** | Cassandra, PostgreSQL | < 1ms latencia, TTL nativo, 1M+ ops/seg |
| ML Framework | **XGBoost + LightGBM Ensemble** | DNN, Random Forest | < 15ms inferencia + SHAP + AUC-PR ≥ 0.90 |
| Serving | **AKS + FastAPI + ONNX** | Azure Functions, App Service | Control fino de scaling, sin cold start |
| Experiment Tracking | **MLflow** | W&B, Comet | Open source, integración nativa Databricks |

### Principios MLOps Aplicados

1. **Reproducibilidad**: Todo versionado — código (Git), datos (Delta Lake), modelos (MLflow Registry)
2. **Automatización**: Pipeline de reentrenamiento 100% automatizado (trigger cada hora)
3. **Monitoreo Continuo**: Evidently AI cada 15 min → trigger automático de reentrenamiento
4. **Degradación Graceful**: Circuit Breaker + fallback a reglas estáticas garantizan 99.99% SLA
5. **Ciclo Cerrado**: Datos → Features → Modelos → Producción → Predicciones → Monitoreo → Datos...

### Métricas de Éxito

| KPI | Baseline Actual | Objetivo Año 1 |
|---|---|---|
| Tasa de detección de fraude | 62% | **≥ 90%** |
| Tasa de falsos positivos | 18% | **≤ 3%** |
| Latencia p99 | Sin definir | **< 200ms** |
| Disponibilidad | Sin definir | **99.99%** |
| Pérdidas por fraude/año | COP $4.200M | **≤ COP $1.260M** |

---

### Stack Tecnológico Completo

| Categoría | Tecnología |
|---|---|
| Cloud Provider | **Microsoft Azure** |
| Event Streaming | **Apache Kafka** (self-managed en AKS) |
| Procesamiento Distribuido | **Apache Spark** (Azure Databricks) |
| Feature Store | **Redis** (online) + **Delta Lake / ADLS Gen2** (offline) |
| ML Frameworks | **XGBoost** + **LightGBM** + **scikit-learn** + **PyCaret** |
| Experiment Tracking | **MLflow** |
| Model Serving | **ONNX Runtime** + **FastAPI** |
| Orquestación Contenedores | **Kubernetes (AKS)** + **Docker** |
| Auto-scaling | **KEDA** (Kafka-based) + **HPA** |
| Monitoreo | **Evidently AI** + **Azure Monitor** + **Grafana** |
| CI/CD | **Azure DevOps** + **GitHub Actions** |
| Seguridad | **Azure AD** + **Key Vault** + **TLS 1.3** + **AES-256** |
| Lenguaje principal | **Python 3.11** |
| Control de versiones | **Git** |

---

*Proyecto desarrollado para: Curso de Procesamiento Distribuido de Datos — Marzo 2026*  
*Caso de uso: PayCol Fintech — Detección de Fraude en Tiempo Real*  
*Documentación completa disponible en: [`MLOPS_CasodeUso.md`](./MLOPS_CasodeUso.md) y [`MLOPS_Diagram.md`](./MLOPS_Diagram.md)*
