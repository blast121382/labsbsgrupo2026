# LAB-05: Resumen de Cierre de Entrega y Verificación de Reproductibilidad

Este documento sintetiza la culminación de los 5 laboratorios del curso **Estrategias de Integración con LLMs** (BSG Institute) y certifica la consistencia entre los ejercicios guiados y el proyecto final individual.

---

## 1. Matriz de Entregables de Laboratorio (35% de la Calificación)

| Sesión / Lab | Tema Evaluado | Evidencias Clave Entregadas | Estado |
|:---:|---|---|:---:|
| **Sesión 2 (Lab 01)** | Contratos, HTTP, Autenticación y Reglas | • Respuesta 200 OK con comparación meteorológica<br>• Rechazo 401 por falta de API Key (`X-API-Key`)<br>• Rechazo 422 por validación estricta de Pydantic<br>• Diagrama explicativo de secuencia | ✅ Completo |
| **Sesión 3 (Lab 02)** | Rendimiento, Streaming SSE y Latencia | • Captura de respuesta completa `/planificar`<br>• Secuencia SSE (`datos`, `texto`, `fin`) `/planificar/stream`<br>• Mediciones comparativas con `medir.py` (`mediciones.json`)<br>• Análisis de TTFT y percepción de usuario | ✅ Completo |
| **Sesión 4 (Lab 03)** | Observabilidad Distribuida con Langfuse | • Ejecución local desacoplada (`LANGFUSE_ENABLED=false`)<br>• Registro estructurado `.jsonl` para DeepEval<br>• Mapeo de traza y spans (API -> Tool -> Clima -> LLM)<br>• Análisis de resiliencia y diagnóstico de fallos | ✅ Completo |
| **Sesión 5 (Lab 04)** | Evaluación Automatizada con DeepEval | • Salida de ejecución didáctica (1.00, 0.00, 1.00)<br>• Casos didácticos en `casos-ejemplo.jsonl`<br>• Captura real de consulta meteorológica<br>• Diagnóstico formal del fallo por alucinación de argumentos y propuesta de mitigación | ✅ Completo |
| **Sesión 6 (Lab 05)** | Empaquetado Reproducible y Despliegue | • Suite completa de pruebas unitarias (`34 passed` en pytest)<br>• Registro de verificación de Docker y manejo de entorno<br>• Auditoría de seguridad de `Dockerfile` y `.dockerignore`<br>• Verificación de aislamiento de credenciales | ✅ Completo |

---

## 2. Relación con el Proyecto Final Individual (40%)

El aprendizaje metodológico desarrollado a lo largo de estos cinco laboratorios sirvió de base directa para la arquitectura del proyecto final personal:

- **Proyecto:** **CriptoAdvisor API** (Servicio Asesor de Criptomonedas basado en CoinGecko y LLM).
- **Repositorio Independiente:** `https://github.com/blast121382/proyfinal2026BsGrupo`
- **Trazabilidad:** Ambos repositorios mantienen estrictamente separada la lógica de negocio y las evidencias, garantizando la reproducibilidad y el cumplimiento de las rúbricas docentes.
