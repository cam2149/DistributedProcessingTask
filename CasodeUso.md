# DESCRIPCIÓN GENERAL
Cada grupo deberá diseñar y presentar una arquitectura de procesamiento distribuido aplicada a un caso de negocio real asignado. El entregable principal es una presentación en PowerPoint que evidencie tanto el dominio técnico como la capacidad de comunicar decisiones de arquitectura.
El proveedor de nube es de libre elección (AWS, Azure, GCP u on-premise con herramientas open source). Lo importante es justificar cada componente seleccionado.
## ESTRUCTURA DE LA PRESENTACIÓN
La presentación debe contener mínimo las siguientes secciones:
1. Portada — Nombre del grupo, caso asignado, integrantes
2. Contexto y problema de negocio — ¿Qué problema resuelve?
3. Requerimientos técnicos — Volumen, velocidad, variedad de datos
4. Arquitectura propuesta — Diagrama completo con todos los componentes
5. Justificación por componente — Por qué se eligió cada servicio/herramienta
6. Clasificación de componentes — IaaS / PaaS / SaaS para cada elemento
7. Modelo de procesamiento — Qué es batch, qué es streaming, qué es paralelo
8. Consideraciones de escalabilidad — Cómo escala la solución ante picos
9. Estimación de costos — Orden de magnitud del costo mensual aproximado
10. Conclusiones y lecciones aprendidas
## CRITERIOS DE EVALUACIÓN (100 puntos)
- **[25 pts] Arquitectura técnica**
- Diagrama claro, componentes apropiados para el caso, flujo de datos coherente de extremo a extremo.
- **[20 pts] Justificación de componentes**
- Cada componente tiene una razón técnica explicada.
- Se comparan al menos 2 alternativas por decisión clave.
- **[20 pts] Clasificación correcta**
- IaaS / PaaS identificados correctamente.
- Batch vs Streaming vs Paralelo bien diferenciados.
- Se justifica por qué se eligió cada modelo de procesamiento.
- **[15 pts] Escalabilidad y resiliencia**
- La arquitectura maneja picos de carga.
- Se menciona tolerancia a fallos y estrategia de recuperación.
- **[10 pts] Estimación de costos**
- Orden de magnitud razonable con fuentes citadas (calculadora del proveedor cloud elegido).
- **[10 pts] Claridad y presentación**
- Diagramas legibles, narrativa coherente, dominio del tema durante la exposición y respuesta a preguntas.
 
 ## RECURSOS RECOMENDADOS
- Calculadora de costos AWS: [https://calculator.aws](https://calculator.aws)
- Calculadora de costos Azure: [https://azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator)
- Calculadora de costos GCP: [https://cloud.google.com/products/calculator](https://cloud.google.com/products/calculator)
- Diagramas de arquitectura: [https://www.diagrams.net](https://www.diagrams.net) (draw.io) o Lucidchart
- Referencia de patrones: [https://aws.amazon.com/architecture](https://aws.amazon.com/architecture)
