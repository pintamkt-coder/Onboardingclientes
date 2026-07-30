# Matriz de Costos y Decisión de Modelos de IA

## Por qué este proyecto (el problema de negocio real)

Hoy, buena parte de los proyectos de Pinta arrancan **de palabra**: se acuerda el alcance, se arma el equipo, se invierte tiempo de desarrollo — y si el cliente se baja antes de firmar nada, ese costo de armado e inversión queda a cargo de Pinta, sin ningún respaldo. Este ecosistema fuerza que **el contrato y el presupuesto siempre se generen, revisen y envíen** como paso obligatorio antes de arrancar a trabajar en un proyecto — no depende de que alguien se acuerde de hacerlo manualmente en medio de la operación diaria. El HITL (revisión humana antes de enviar) mantiene el control de calidad sin perder la automatización del resto del proceso.

## Justificación del stack (más allá del costo por token)

| Herramienta | Por qué se eligió |
|---|---|
| **Google Sheets** (no Airtable) | Pinta ya paga la cuenta de Google Workspace — Sheets no suma costo adicional. Airtable es pago y, para una tabla de control interno sin la complejidad de vistas/relaciones avanzadas que Airtable ofrece, hubiera sido gasto sin beneficio real. Además, este registro es 100% interno (no lo ve ningún cliente), así que no hace falta la interfaz más pulida de Airtable. |
| **Make** | Se eligió por el **tipo de proyecto**, no solo por costo: necesita orquestar Google Sheets + Drive + Docs + Gmail + un modelo de IA + un punto de espera humana (HITL) en un solo flujo visual, con manejo de errores por paso. Make cubre esa integración multi-servicio sin desarrollo custom. |
| **Claude (Anthropic)** | Pinta ya tiene la herramienta contratada para otras tareas del equipo, con resultados probados. Reutilizarla acá evita sumar un proveedor de IA nuevo solo para esta tarea, y da calidad de redacción consistente a un costo que, como se detalla abajo, es prácticamente irrelevante al volumen de uso real. |

## Tarea de IA en el flujo

**Matriz aplicada a la única tarea con LLM del flujo: redacción del contrato (1 llamada por cliente).**

Este ecosistema tiene **una única tarea que requiere un modelo de lenguaje**: la redacción del contrato a partir de los datos del cliente (módulo "Anthropic Claude — Create a Prompt" en Escenario A). No hay clasificación, triage, ni ningún otro paso que use IA.

Estimación de uso por ejecución (un contrato = una llamada):

| Concepto | Estimado |
|---|---|
| Input (system prompt + plantilla + datos del cliente) | ~2.000 tokens |
| Output (contrato redactado completo) | ~1.500 tokens |
| Frecuencia | 1 vez por cliente nuevo (evento, no recurrente) |

## Comparativa de modelos

Precios oficiales por millón de tokens (input / output), a julio 2026:

| Modelo | Input | Output | Costo estimado por contrato* | Uso recomendado |
|---|---|---|---|---|
| **Claude Haiku 4.5** | $1.00 | $5.00 | ~$0,01 | Clasificación, extracción, tareas simples |
| **Claude Sonnet (4.6/5)** ← elegido | $2-3 | $10-15 | ~$0,02–0,03 | Redacción coherente, instrucciones complejas, mejor relación calidad/precio |
| Claude Opus | $5.00 | $25.00 | ~$0,05 | Razonamiento profundo, no justificado para este caso |
| GPT-4o | $2.50 | $10.00 | ~$0,02 | Alternativa de calidad similar a Sonnet |
| GPT-4o-mini | $0.15 | $0.60 | ~$0,001 | Tareas masivas de bajo riesgo |

*Costo estimado = (input tokens × precio input) + (output tokens × precio output), sobre 1M tokens.

## Por qué Claude Sonnet y no GPT-4o-mini

GPT-4o-mini es hasta **20-30 veces más barato** por token que Sonnet. Aun así, se descartó para esta tarea puntual por 3 motivos:

1. **Volumen bajísimo.** Pinta onboarda clientes nuevos, no en volumen masivo — estimando 10-20 contratos por mes, la diferencia de costo entre el modelo más caro y el más barato de la tabla es de **centavos de dólar al mes**, no dólares. Optimizar el costo acá no mueve el presupuesto de la empresa.
2. **El output es un documento legal/comercial real** que el cliente firma. Un error de coherencia, un dato mal ubicado, o una cláusula ambigua tiene costo de negocio (reputacional y potencialmente legal) muy superior a cualquier ahorro de céntimos por request. Se prioriza calidad y consistencia de redacción sobre el precio por token.
3. **Sonnet ofrece la mejor relación calidad/precio** para generación de texto largo y estructurado (contratos), superando a Haiku en coherencia sin llegar al costo de Opus, que sería sobredimensionado para esta tarea.

**Conclusión:** para tareas de bajo volumen y alto impacto por error, el costo por token deja de ser la variable relevante — lo que importa es la confiabilidad del output. Sonnet es el punto óptimo en esta curva para este caso de uso específico.

### Ahorro operativo si se migrara a un modelo más barato

Sobre ~15 contratos/mes:

- **Con Sonnet (elegido):** ~$0,43/mes en total.
- **Si migráramos a Haiku:** ~$0,14/mes → ahorro de ~$0,29/mes (~67% menos).
- **Si migráramos a GPT-4o-mini:** ~$0,02/mes → ahorro de ~$0,41/mes (~95% menos).

El ahorro porcentual es grande, pero en términos absolutos es de centavos de dólar al mes. Frente a eso, el costo de un contrato mal redactado (relectura, corrección, demora en el onboarding, o peor, un error que llegue al cliente) es varios órdenes de magnitud mayor que cualquiera de esos ahorros. Por eso se prioriza calidad sobre el ahorro de $/token en esta tarea puntual.

## Batch API — por qué no se usa en este flujo

La Batch API de Anthropic (y de OpenAI) da un **50% de descuento** en input y output, pero a cambio de que el procesamiento no sea en tiempo real (hasta 24hs de demora). Este flujo **no es compatible con Batch** porque:

- El contrato se genera al toque de que el cliente completa el formulario — es un evento en tiempo real, no un lote acumulado.
- El admin espera recibir el mail de revisión en minutos, no al día siguiente.

**Dónde sí aplicaría a futuro:** si Pinta necesitara re-generar contratos en lote (por ejemplo, migrar 200 clientes viejos a una plantilla nueva de una sola vez), ahí la Batch API tendría sentido — se armaría un escenario separado, disparado manualmente, que no depende de la latencia del onboarding en vivo.

## Costo mensual estimado del sistema completo

Con Sonnet y ~15 contratos/mes: **menos de USD $0,50/mes** en consumo de IA. El costo real del sistema está dominado por las operaciones de Make (créditos por ejecución), no por el modelo de lenguaje elegido.
