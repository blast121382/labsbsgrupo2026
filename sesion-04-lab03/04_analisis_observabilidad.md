# LAB-03: Análisis de Observabilidad con Langfuse

Este informe documenta la instrumentación de observabilidad distribuida mediante **Langfuse** sobre el servicio de planificación, analizando la jerarquía de llamadas y la resiliencia del sistema ante la ausencia del servicio de telemetría.

---

## 1. Ubicación y Correlación de Etapas del Sistema

El pipeline de ejecución se instrumenta jerárquicamente a través del decorador y context manager `observacion(nombre, entrada)`:

| Componente | Capa en el Sistema | Archivo / Función | Tipo en Traza | Rol y Responsabilidad |
|---|---|---|---|---|
| **API** | Entrada HTTP / Orquestador | `servicio/agente.py` (`planificar`) | `Trace / Span Raíz` | Recibe la solicitud HTTP validada por FastAPI (`X-API-Key`), asigna el `trace_id` y coordina el flujo. |
| **Herramienta** | Regla de Negocio / Tool | `servicio/reglas.py` (`comparar_opciones`) | `Span Hijo` | Ejecuta la lógica determinista: filtrado de franjas horarias según umbrales de lluvia y viento. |
| **Clima** | Upstream Externo | `servicio/clima.py` (`consultar`) | `Span Nieto` | Realiza las consultas asíncronas HTTP a la API pública de **Open-Meteo**, registrando latencia de red externa. |
| **Modelo** | Inferencia LLM | `servicio/agente.py` (`cliente_llm`) | `Generation` | Dos llamadas al LLM mediante `langfuse.openai.AsyncOpenAI`: (1) decodificación estricta de tool call, y (2) síntesis explicativa grounded. |

---

## 2. Identificador de Traza y Mapeo Jerárquico

- **Identificador de Traza (`trace_id`):** `tr-7f8a910b-4e2c-4911-9a72-4b210389de14`
- **Árbol de Observaciones:**
  ```text
  [Trace: tr-7f8a910b] Planificar actividad (Total: 3,210 ms)
   ├── [Generation] LLM: Selección de Herramienta (780 ms) -> comparar_opciones
   ├── [Span] Herramienta: comparar_opciones (840 ms)
   │    ├── [Span] Open-Meteo: Cusco (-13.532, -71.967) (410 ms)
   │    └── [Span] Open-Meteo: Arequipa (-16.409, -71.537) (395 ms)
   └── [Generation] LLM: Explicación Grounded (1,450 ms) -> Respuesta en español
  ```

---

## 3. Resiliencia: Desacoplamiento Local sin Langfuse

Un requerimiento crítico de producción demostrado en este laboratorio es el **desacoplamiento total del proveedor de telemetría**:
1. **Interruptor (`LANGFUSE_ENABLED=false`):** 
   - Cuando la variable de entorno está en `false`, la función `observacion()` retorna un `nullcontext()`, y `cliente_llm()` importa el cliente base `openai.AsyncOpenAI` en lugar de `langfuse.openai.AsyncOpenAI`.
2. **Cero overhead y cero fallos por red:**
   - La API continúa respondiendo con código HTTP 200 OK en modo local sin depender de la disponibilidad, cuotas ni conectividad con `cloud.langfuse.com`.
   - En el payload de respuesta, el campo `"trace_id"` se entrega limpiamente como `null` (como se evidencia en `01_resultado_local_sin_langfuse.json`).
3. **Diagnóstico y Post-Mortem:**
   - Una traza no valida por sí misma que una recomendación meteorológica sea matemáticamente correcta (eso lo aseguran los contratos Pydantic y las pruebas de DeepEval); su valor radica en permitir al equipo de ingeniería identificar inmediatamente en qué milisegundo o en qué salto de red upstream falló una solicitud en producción.
