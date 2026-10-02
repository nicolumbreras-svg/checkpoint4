# 🚗 Sistema Integrado de Atención y Ventas Multicanal para Concesionario (n8n)

Este repositorio contiene la arquitectura, flujos de trabajo y configuraciones necesarias para desplegar una solución integral de automatización basada en **n8n** y **Modelos de Lenguaje (LLMs)**. El sistema atiende consultas de clientes a través de **Telegram** y **Gmail**, enrutando automáticamente las intenciones hacia subflujos especializados, gestionando agenda, stock, aprobación humana (HITL) y sincronización con **Salesforce** y **Airtable**.

---

## 📐 Arquitectura General

El sistema implementa un patrón **Router / Worker Agents** coordinado por un orquestador central (Módulo 4) que delega tareas específicas a subflujos especializados (Módulos 1, 2 y 3).

                      [Cliente: Telegram / Gmail]
                               │
                               ▼
                  [Módulo 4: Orquestador Central]
                               │
                  [Persistencia en Airtable]
                               │
                 [Clasificación IA - Cohere]
                               │
     ┌─────────────────────────┼─────────────────────────┐
     ▼                         ▼                         ▼
[Subflujo 1]                [Subflujo 2]              [Subflujo 3]
Catálogo Usados             Agendamiento              Datos Empresa
(Google Sheets)           (Google Calendar)           (Google Sheets)
└─────────────────────────┼─────────────────────────┘
                          │
                [Respuesta Normalizada]
                          │
             [Detección de Canal de Entrada]
                  ├── Telegram ──> [Aprobación HITL en Slack] ──> [Telegram Message]
                  └── Gmail    ──> [Borrador Gmail (DRAFT)]
             │
             [¿Turno Agendado? == Sí]
                │
                [Upsert Oportunidad en Salesforce]
             │
             [Log Observabilidad + Compresión Memoria (Cohere) + Airtable]


---

## 🧩 Descripción de Módulos y Subflujos

### 🎛️ Módulo 4: Orquestador Central (`checkpoint4_nicolas_lumbreras`)
Es el núcleo del sistema. Se encarga de:
- **Entrada Multicanal**: Recepción de eventos desde Telegram y Gmail.
- **Normalización**: Estandarización de variables (`canal`, `id`, `mensaje`, `cliente`, `subject`).
- **Persistencia e Historial**: Consulta e historización de clientes y resúmenes conversacionales en **Airtable**.
- **Clasificación Semántica**: Evaluación con un agente de IA (`Cohere Chat Model`) para determinar la intención (`CATALOGO`, `AGENDAMIENTO`, `DATOS` o `FALLBACK`).
- **Control HITL (Human-in-the-Loop)**: Aprobación interactiva en **Slack** previa al envío del mensaje en Telegram.
- **Sincronización CRM**: Actualización de oportunidades en **Salesforce** tras un agendamiento exitoso.
- **Mantenimiento de Memoria**: Compresión y resumen del historial conversacional mediante IA cuando se supera el umbral de 5 mensajes.

---

### 📦 Subflujo 1: Módulo de Catálogo (`modulo3_workflow1`)
- **Función**: Agente experto en la consulta del stock de vehículos usados.
- **Integración**: Se conecta a una planilla de **Google Sheets** con los datos actualizados de vehículos.
- **Modelo de IA**: `Google Gemini Chat Model` (`gemini-3.5-flash-lite`).
- **Salida**: Recomienda hasta 5 opciones identificadas con letras (a-e). Si el usuario selecciona un auto, emite la marca de control `AUTO SELECCIONADO: [Vehículo]` para actualizar el estado del cliente en Airtable.

---

### 📅 Subflujo 2: Módulo de Agendamiento (`modulo3_workflow2`)
- **Función**: Coordinador de citas de prueba de manejo o atención comercial.
- **Integración**: Ejecuta un nodo de código (`generar_horarios_disponibles`) que calcula 5 turnos hábiles disponibles y utiliza la herramienta **Google Calendar Tool** para registrar el evento.
- **Modelo de IA**: `Google Gemini Chat Model` (`gemini-3.5-flash-lite`).
- **Salida**: Genera el turno en la agenda y emite la etiqueta `AGENDADO DD/MM/YYYY HH:mm`. Esta etiqueta le indica al Módulo 4 que debe disparar la actualización de la Oportunidad en **Salesforce**.

---

### ℹ️ Subflujo 3: Módulo de Datos de la Empresa (`modulo3_workflow3`)
- **Función**: Recepcionista digital enfocado en información institucional y comercial.
- **Integración**: Consulta la pestaña *"Datos de la Empresa"* en **Google Sheets** (dirección, horarios, responsables, formas de pago).
- **Modelo de IA**: `Google Gemini Chat Model` (`gemini-3.5-flash-lite`).
- **Salida**: Proporciona respuestas precisas y estructuradas sobre la operación del concesionario.

---

## ⚙️ Ciclo de Vida de una Interacción (Paso a Paso)

1. **Recepción del Mensaje**: El flujo es activado por un webhook de Telegram o una llegada de correo en Gmail.
2. **Validación de Filtro**: Se evalúa que no sea una respuesta automática o rebote (`No-reply / Auto-reply`).
3. **Carga de Contexto**: Se busca al cliente en **Airtable**. Si es un cliente nuevo, se crea un registro con estado `NUEVO`. Si existe, se recupera el historial resumido y el conteo de interacciones.
4. **Clasificación y Enrutamiento**: El Agente Clasificador (Cohere) evalúa la intención y dispara el subflujo correspondiente mediante el nodo `Execute Workflow`.
5. **Ejecución del Subflujo**: El agente interno (Gemini) resuelve la consulta usando la herramienta apropiada (**Google Sheets** o **Google Calendar**).
6. **Formateo de Respuesta**: El Módulo 4 recibe la respuesta, extrae etiquetas clave (`AUTO SELECCIONADO`, `AGENDADO`) y determina el nuevo estado de la conversación.
7. **Despacho Multicanal**:
   - **Telegram**: Envía una notificación a **Slack** para aprobación humana (Aprobar / Rechazar). Al ser aprobada, se envía el mensaje al cliente.
   - **Gmail**: Genera automáticamente un borrador (`DRAFT`) para revisión.
8. **Actualización CRM**: Si se agendó un turno (`TURNO AGENDADO`), se ejecuta un *upsert* en **Salesforce** creando o actualizando la oportunidad en la etapa `"Meet & Present"`.
9. **Observabilidad y Compresión de Memoria**:
   - Envía un correo de monitoreo a supervisión (`Log de Observabilidad`).
   - Si el contador supera los 5 mensajes, invoca a Cohere para resumir la conversación en JSON, limpia el historial detallado y guarda el resumen consolidado en Airtable.

---

## 🛠️ Tecnologías y Herramientas

- **Motor de Automatización**: n8n (Self-hosted / Cloud)
- **Modelos de IA**:
  - Cohere Chat Model (Clasificación de intenciones y compresión de memoria)
  - Google Gemini Chat Model (`gemini-3.5-flash-lite` para agentes worker)
- **Plataformas de Mensajería**: Telegram Bot API, Gmail API, Slack API (HITL)
- **Bases de Datos & CRM**: Airtable API, Salesforce API
- **Productividad**: Google Sheets API, Google Calendar API

---

## 📋 Requisitos Previos e Instalación

1. **n8n**: Instancia de n8n operativa (versión v1.0+ recomendada).
2. **Credenciales en n8n**:
   - Credenciales de Telegram Bot & Gmail API.
   - Credenciales de Slack App / Bot.
   - Credenciales de Airtable API Key / Personal Access Token.
   - Credenciales de Salesforce OAuth2.
   - Credenciales de Google Console (Google Sheets & Google Calendar).
   - API Keys de Cohere y Google Gemini.
3. **Importación de Flujos**:
   - Importar el archivo `checkpoint4_nicolas_lumbreras.json` en n8n.
   - Importar los subflujos `modulo3_workflow1.json`, `modulo3_workflow2.json` y `modulo3_workflow3.json`.
   - Verificar y vincular los IDs de los nodos `Execute Workflow` en el Módulo 4.
