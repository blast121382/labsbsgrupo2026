# LAB-04: Evaluación Automatizada con DeepEval (Tool Correctness y G-Eval)

Este informe documenta la sesión 5 sobre la evaluación formal de agentes que utilizan llamadas a herramientas (Tool Calling) y generación de respuestas explicativas, utilizando el framework **DeepEval**.

---

## 1. El Flujo de Evaluación en Cuatro Pasos

El script `evaluacion/evaluacion.py` implementa el estándar metodológico de evaluación de sistemas LLM:

1. **Lectura y Validación de Contratos (`cargar_registros`):**
   - Carga el archivo `.jsonl` donde cada línea es un objeto JSON validado contra el esquema Pydantic `Registro`. Garantiza que campos como `tools_called`, `expected_tools` y `context` tengan tipos válidos antes de iniciar la medición.
2. **Construcción del Caso de Prueba (`crear_caso`):**
   - Transforma el registro en una instancia oficial de `LLMTestCase` de DeepEval, mapeando `input`, `actual_output`, `expected_output`, `context`, y las instancias de `ToolCall` con sus parámetros de entrada.
3. **Medición Determinista y con Juez (`evaluar`):**
   - **Ruta Determinista (`ToolCorrectnessMetric`):** Evalúa con `should_exact_match=True` y `threshold=1.0` si la herramienta invocada y sus parámetros (`INPUT_PARAMETERS`) coinciden exactamente con la referencia esperada, utilizando un mock local `ModeloNoUsado(model="sin-llm")` sin costo ni llamadas al LLM.
   - **Ruta Semántica (`GEval` con Juez):** Evalúa la fidelidad de la explicación contra los datos del contexto mediante pasos de razonamiento fijos (`PASOS_JUEZ`).
4. **Interpretación y Veredicto:**
   - Emite el puntaje cuantitativo (`score`), el veredicto booleano (`APROBÓ` o `NO APROBÓ`) y la justificación explicativa (`reason`).

---

## 2. Resultados del Conjunto Didáctico Oficial

Al ejecutar `python evaluacion/evaluacion.py --archivo evaluacion/casos-ejemplo.jsonl`, se obtuvieron los siguientes resultados:

| Caso de Prueba | Tipo de Caso | Métrica | Puntuación | Resultado | Veredicto |
|---|---|---|:---:|:---:|:---:|
| `ejemplo-aprobado` | Consulta válida de clima | Tool Correctness | **1.00** | APROBÓ | ✅ Válido |
| `ejemplo-argumentos-incorrectos` | Caso intencional negativo | Tool Correctness | **0.00** | NO APROBÓ | ❌ Rechazado |
| `ejemplo-aclaracion-sin-herramienta` | Solicitud incompleta | Tool Correctness | **1.00** | APROBÓ | ✅ Válido |

**Veredicto Global:** **NO APROBÓ** (debido al fallo controlado en el caso negativo).

---

## 3. Diagnóstico del Fallo en `ejemplo-argumentos-incorrectos`

### ¿Qué ocurrió?
En el caso `ejemplo-argumentos-incorrectos`:
- **Entrada del usuario:** `"Compara una salida de cuatro horas el 2026-09-13 entre Lima y Chosica."`
- **Herramienta Esperada (`expected_tools`):**
  ```json
  {
    "name": "comparar_opciones",
    "input_parameters": {
      "fecha": "2026-09-13",
      "duracion_horas": 4
    }
  }
  ```
- **Herramienta Realmente Invocada por el Modelo (`tools_called`):**
  ```json
  {
    "name": "comparar_opciones",
    "input_parameters": {
      "fecha": "2026-09-14",
      "duracion_horas": 2
    }
  }
  ```

### Razón del Fallo:
El modelo alucinó la fecha desplazándola en un día (`2026-09-14`) y redujo arbitrariamente la duración a la mitad (`2` horas en lugar de `4`). Dado que `ToolCorrectnessMetric` opera con `should_exact_match=True`, cualquier discrepancia exacta en los parámetros produce un puntaje de `0.00` y rechaza el test.

### Corrección y Defensas Arquitectónicas Propuestas:
1. **Defensa por Esquema Estricto (`strict: True`):**
   - Configurar `strict: True` en la definición de la herramienta para OpenAI/Groq para forzar al decodificador constrained a respetar los tipos.
2. **Validación Determinista con Pydantic (`extra="forbid"`):**
   - Parsear los argumentos generados por el LLM mediante una clase `ArgumentosComparar(BaseModel)` con validaciones de rango.
3. **Mecanismo de Corrección por Reintento (Self-Correction Loop):**
   - Si los parámetros extraídos discrepan de la solicitud estructurada que ya posee el backend, el servicio no debe ejecutar la herramienta errónea ni inventar datos; debe rechazar la llamada o reintentar inyectando el error como mensaje de sistema.
