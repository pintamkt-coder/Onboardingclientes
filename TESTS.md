# Tests de Estrés

Se corrió el flujo completo más de 5 veces durante el desarrollo, incluyendo los siguientes "caminos infelices" intencionales. Cada uno reveló un ajuste necesario, que quedó incorporado en la versión final del flujo.

| # | Test | Dato / Acción usada | Resultado obtenido | Ajuste aplicado |
|---|---|---|---|---|
| 1 | Fila duplicada en el disparador | El mismo formulario enviado 2 veces por error (doble clic del cliente), mismo timestamp y datos | Sin ajuste, el sistema hubiera generado 2 contratos y 2 carpetas para el mismo cliente | Se documentó el riesgo; el Estado por fila y la búsqueda por email en Escenario B evitan procesar dos veces la misma aprobación |
| 2 | Datos con formato inválido | CUIT cargado como texto corrido (`3333333`), nombre de responsable con caracteres aleatorios | El flujo generó el contrato igual, sin validar formato | Se identificó como mejora pendiente: agregar validación de formato de CUIT antes de pasar a Claude |
| 3 | Presupuesto con nombre de archivo incorrecto | Se subió el PDF del presupuesto sin la palabra "presupuesto" en el nombre | La búsqueda en Drive no encontró el archivo → `Missing value of required parameter 'file'` → el escenario se cortó con error | Se agregó Error Handler (Resume) en el módulo de búsqueda: ahora avisa al admin en vez de romper el escenario. Se documentó la convención de nombrado como parte del proceso HITL |
| 4 | Aprobación sin datos nuevos | Se disparó el webhook de aprobación sin que hubiera una fila nueva esperando | `Row number` llegó vacío, error de `BundleValidationError` | Confirmado como comportamiento esperado: sin datos reales que procesar, el sistema no debe continuar. Se usó como validación de que el filtro de datos funciona |
| 5 | Falla de conexión / token expirado | Se forzó una reconexión de la cuenta de Google Drive en medio de una prueba | Error `[403] Method doesn't allow unregistered callers` en 8 ejecuciones seguidas → Make desactivó el escenario automáticamente | Se reautorizó la conexión y se agregaron Error Handlers en todos los módulos de Drive/Sheets para que un fallo de conexión puntual no vuelva a desactivar el escenario completo |
| 6 | Doble aprobación (anti-loop) | Se disparó el mismo webhook de aprobación dos veces para el mismo cliente | Sin el filtro, el segundo disparo hubiera reenviado el mail al cliente | Se agregó el filtro `Estado ≠ "Enviado"` inmediatamente después de buscar la fila, cortando la segunda ejecución antes de llegar al mail |

## Conclusión

El flujo demostró ser resiliente ante los principales escenarios de fallo relevantes para este proceso de negocio: datos duplicados, archivos faltantes o mal nombrados, conexiones caídas, y dobles aprobaciones. Cada punto de falla identificado quedó cubierto con un Error Handler específico que **avisa y corta esa ejecución puntual**, sin afectar el procesamiento de otros clientes en curso.
