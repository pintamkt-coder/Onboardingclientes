# Ecosistema de Automatización IA — Onboarding de Clientes (Pinta)

Entrega final del curso de Automatización IA. Sistema que automatiza de punta a punta el proceso de **onboarding de nuevos clientes de Pinta**, desde que se cierra un presupuesto hasta que el cliente recibe el contrato y el presupuesto firmados por mail, con un punto de validación humana en el medio.

## Caso de uso

Cuando Pinta cierra un presupuesto nuevo con un cliente, se le envía un Google Form para que cargue sus datos (razón social, CUIT, domicilio, email, servicios contratados, etc.). A partir de esa carga, el sistema:

1. Redacta el contrato automáticamente con IA.
2. Crea la carpeta del cliente en Drive y guarda el contrato ahí.
3. Notifica al equipo de administración para que revise el contrato y suba el presupuesto.
4. Espera la aprobación humana (HITL) antes de contactar al cliente.
5. Una vez aprobado, envía el contrato y el presupuesto al cliente por mail, y deja registro de todo en la base de datos.

## Stack utilizado

| Componente | Herramienta |
|---|---|
| Orquestador | Make (2 escenarios) |
| Base de datos / memoria | Google Sheets (ver nota más abajo) |
| Procesamiento IA | Anthropic Claude (redacción del contrato) |
| Canal de salida | Gmail |
| Documentos | Google Docs / Google Drive |

> **Nota sobre la base de datos:** la consigna sugiere Airtable o Notion. Se utilizó **Google Sheets** en su lugar, con aprobación explícita de Ticher para esta entrega. Cumple el mismo rol de memoria y registro del sistema: campos de estado, timestamps por etapa, y relación 1:1 fila-cliente.

## Arquitectura

Diagrama completo en [`/pdf/Diagrama_Arquitectura_Onboarding_Pinta_FINAL.pdf`](./pdf/Diagrama_Arquitectura_Onboarding_Pinta_FINAL.pdf).

El sistema está dividido en **2 escenarios de Make** conectados por un punto de validación humana:

### Escenario A — "Onboarding clientes"
`Google Form → Watch New Rows → Get Content (plantilla) → Claude (redacta contrato) → Create Folder → Create Document (contrato) → Update Row (Estado="Pendiente Revisión") → Gmail a admin`

El mail al admin incluye 3 links: abrir la carpeta del cliente, editar el documento del contrato, y un botón **"Aprobar y enviar al cliente"** que dispara un webhook con los datos necesarios (`docId`, `clienteEmail`, `razonSocial`, `nombreContacto`, `folderId`).

### HITL (Human-in-the-loop)
El administrador:
1. Abre el contrato generado y lo revisa/edita si hace falta.
2. Sube el presupuesto en PDF a la carpeta del cliente — **el nombre del archivo debe contener la palabra "presupuesto"**, es la convención que usa el sistema para encontrarlo automáticamente.
3. Hace clic en "Aprobar y enviar al cliente".

Ningún mail sale al cliente sin este paso.

### Escenario B — "Aprobación de contratos onboarding"
`Webhook → Search Rows (busca la fila por email) → Filtro anti-loop (Estado ≠ "Enviado") → Search Drive (busca el presupuesto) → Download presupuesto → Download contrato (export a PDF) → Gmail al cliente (2 adjuntos) → Update Row (Estado="Enviado")`

## Manejo de errores

Cada paso crítico de ambos escenarios tiene su propio **Error Handler (Resume)** independiente, para que una falla puntual (ej. el admin todavía no subió el presupuesto, o falla la API de Drive) no tumbe el escenario entero ni bloquee el procesamiento de otros clientes:

- Sheet no encontrado por email → avisa al admin.
- Presupuesto no encontrado en la carpeta → avisa al admin que falta subirlo.
- Falla la descarga del presupuesto o del contrato → avisa al admin + registra en `Log_Error`.
- Falla el envío del mail al cliente → `Estado="Error Envío"` + `Log_Error`.

Además, un **filtro anti-loop** (`Estado ≠ "Enviado"`) evita que un doble clic en el botón de aprobación le reenvíe el mail al cliente dos veces.

## Base de datos (Google Sheets)

Hoja `NC`, con las columnas del formulario (A-L) más las columnas de control del sistema:

| Columna | Campo |
|---|---|
| M | ESTADO (Pendiente / Pendiente Revisión / Aprobado / Enviado / Error IA / Error Drive / Error Envío) |
| N | CONTRATO_URL |
| O | Carpeta_URL |
| P | Presupuesto_URL |
| Q | Fecha_Generado |
| R | Fecha_Aprobado |
| S | Fecha_Enviado |
| T | Log_Error |

**Link de solo lectura:** `[https://docs.google.com/spreadsheets/d/1iICa0Oi4CGmxPSqJIAoqtXQ9R0bU1TeryK-f9wvE6vA/edit?gid=1797733805#gid=1797733805]`

## Contenido del repositorio

```
├── pdf/
│   └── Diagrama_Arquitectura_Onboarding_Pinta_FINAL.pdf
├── json/
│   ├── escenario_A_onboarding.json
│   └── escenario_B_aprobacion.json
├── screenshots/
│   └── (capturas de evidencia del flujo funcionando)
├── TESTS.md
└── README.md
```

## Tests de estrés

Ver [`TESTS.md`](./TESTS.md) para el detalle de los 5+ tests realizados, incluyendo los "caminos infelices" (datos incompletos, archivos con nombre incorrecto, filas duplicadas, conexiones caídas).

## Video demo

`[https://drive.google.com/file/d/1-xfWRr7HvD6pJ7ZjnnjxwLioatCgchYR/view?usp=drive_link]`

## Equipo

Pinta — Sistemas Digitales & IA.
