# LAB-02: Análisis de Rendimiento y Comparativa Streaming vs Respuesta Completa

Este documento resume los resultados obtenidos al ejecutar el script de benchmark `medir.py --repeticiones 3` sobre el servicio de planificación climática implementado en el **Lab 02**.

---

## 1. Resumen de Métricas Registradas

| Muestra | Endpoint | TTFT (Primer Texto) | Tiempo Total | Estado | Comentario / Incidencia |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **1** | `/planificar` (completa) | N/A (bloqueante) | **3.01 s** | OK | Espera total hasta completar el payload |
| **1** | `/planificar/stream` (SSE) | **2.59 s** | **2.95 s** | OK | Entrega inmediata de `datos` + streaming de texto |
| **2** | `/planificar` (completa) | N/A (bloqueante) | **3.55 s** | OK | Variabilidad inherente a la red externa |
| **2** | `/planificar/stream` (SSE) | **3.23 s** | **3.65 s** | OK | Percepción progresiva para el usuario |
| **3** | `/planificar` (completa) | N/A (bloqueante) | **3.45 s** | OK | Ejecución determinista y LLM completadas |
| **3** | `/planificar/stream` (SSE) | N/A | N/A | **FALLO** | Excepción `ValueError` en deserialización / timeout de socket upstream |

---

## 2. Tres Observaciones Clave: Primer Texto frente a Finalización

### Observación 1: El Streaming optimiza la *percepción de espera* (TTFT), no el *tiempo total de cómputo*
Como revelan las muestras 1 y 2, el tiempo total transcurrido para completar la solicitud en streaming (~2.95 s y ~3.65 s) es prácticamente idéntico al de la respuesta completa (~3.01 s y ~3.55 s). Esto se debe a que:
- Ambos caminos ejecutan exactamente las mismas operaciones computacionales previas (validación de contrato Pydantic, fetch HTTP a Open-Meteo y cálculo determinista de reglas).
- El LLM genera la misma cantidad de tokens en ambos casos.
- Por tanto, **el streaming no acelera la inferencia ni la red**, sino que traslada la entrega del primer feedback visual al usuario al segundo 2.59 s en lugar de obligarlo a ver un spinner en blanco durante más de 3 segundos.

### Observación 2: El evento inicial `datos` desacopla la lógica determinista del LLM
En la arquitectura SSE implementada, el primer evento emitido no es un token del LLM, sino el evento estructurado `datos`. Esto significa que la interfaz gráfica puede renderizar inmediatamente las tarjetas de destino, temperaturas y puntuaciones cuantitativas en cuanto la regla de negocio termina, mientras el LLM continúa generando su explicación textual en segundo plano. La experiencia de usuario pasa de ser síncrona/bloqueante a ser reactiva.

### Observación 3: Manejo de errores y desconexiones a mitad de flujo (Streaming Error Handling)
En la repetición 3 de la ruta en streaming se registró una anomalía (`ValueError`). Cuando una llamada síncrona falla antes de responder, el servidor puede emitir un código de estado HTTP 4xx o 5xx limpio. Sin embargo, en un flujo SSE que ya comenzó a transmitir (HTTP 200 con cabeceras `text/event-stream`), el código HTTP no se puede modificar a posteriori:
- La implementación debe emitir un evento formal `{"evento": "error", "mensaje": "..."}` y cerrar la conexión sin emitir nunca el evento `fin`.
- El cliente frontend debe estar programado para escuchar tanto el canal de datos como el evento de error en el `EventSource`.

---

## 3. Conclusión Arquitectónica
Para sistemas de producción que integran agentes y LLMs, **la respuesta en streaming debe ser la norma para clientes interactivos (web/móvil)** debido a la reducción del TTFT y la mitigación de abandonos por impaciencia del usuario. La **respuesta completa JSON** se reserva para integraciones máquina a máquina (M2M / ETLs) donde un parseo directo sin streaming sea más conveniente.
