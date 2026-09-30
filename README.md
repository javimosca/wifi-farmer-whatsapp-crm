# The WiFi Farmer

> Conecta usuarios. Cosecha clientes.

Plataforma de captación omnicanal, CRM y WhatsApp Marketing que captura leads desde portales WiFi cautivos (sucursales, puntos de acceso y comercios adheridos), los cruza contra el CRM / E-commerce (Tiendanube / ERP) y los activa con campañas salientes de WhatsApp mediante la **API Oficial de Meta (Cloud API)**. Incluye una bandeja única de atención multiagente y una Ficha 360° de cada persona.

---

## Qué hace

| Módulo | Qué resuelve |
|---|---|
| **Captación WiFi** | Recibe cada login del portal cautivo (Easy Auth), identifica la antena (AP) por dirección MAC, cruza a la persona contra el CRM/Tienda y la guarda en `unified_contacts` con su historial de visitas. |
| **SCV / Contactos** | Vista única de cada contacto (*Single Customer View*): estado comercial, servicios/compras activas, recurrencia de visitas y antena de origen. |
| **Campañas** | Envío de plantillas de WhatsApp aprobadas a segmentos de dos orígenes: **Hotspots** (leads captados en WiFi) y **Clientes Activos** (e-commerce y cross-sell). Envío asíncrono en segundo plano con barra de progreso en tiempo real. |
| **Plantillas** | Alta, consulta de estado de aprobación y baja de plantillas directo en Meta Graph API, con soporte de imagen en el encabezado y botón de acción. |
| **Redirector `/go/{codigo}`** | El botón de cada plantilla pasa por el acortador propio `/go/{CODIGO}`, registra el clic real (filtrando bots por `ip_hash`) y redirige al chat con atribución de campaña (`CAMP-{CODIGO}`). |
| **Mensajes (Bandeja)** | Bandeja unificada por persona (`phone10`) que centraliza respuestas de campañas y consultas de tienda online. Permite responder en texto libre dentro de la ventana de 24 h, disparar plantilla de seguimiento fuera de ventana y tipificar el cierre. |
| **Ficha 360°** | Panel lateral global (`Ctrl+K`) con datos comerciales, visitas a hotspots, historial de WhatsApp, compras en tienda y línea de tiempo unificada. |
| **Avisos en la PC** | Notificaciones nativas Web Push (VAPID) cuando escribe una persona, incluso con el panel cerrado en el navegador. |
| **Antenas y Comercios** | Alta de Access Points (APs) por MAC, asignación de comercio, ubicación y reportes de captación por punto físico (`/comercio/[slug]`). |

---

## Arquitectura

```text
[Portal Cautivo · Easy Auth]          [ERP / Tiendanube Sync]       [Meta Cloud API (v23.0)]
 POST /webhook/easyauth                POST /whatsapp/crosssell/     Webhooks de mensajes
 (X-Webhook-Secret)                    ingesta (por DNI / Email)     y estados de entrega
        │                                     │                             │
        ▼                                     ▼                             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Backend FastAPI (Python 3.11+ · asyncpg · httpx)                                │
│  main.py · whatsapp_routes.py · whatsapp_provider.py · wa_entrantes · bandeja.py │
│  ficha.py · web_push.py · db_ddl.py                                              │
│                                                                                  │
│  Captación: AP por MAC → Matching CRM/Tienda → Clasificación en funnel_stage     │
│  Campañas:  Segmento → Cadencia/Bajas → send_template (background) → Progreso    │
│  Entrantes: Opt-out (BAJA) → Clasificación por reglas → Bandeja → Aviso Push     │
└──────────────────────────────────────────────────────────────────────────────────┘
        │                       │                              │
        ▼                       ▼                              ▼
   PostgreSQL            CRM / E-commerce API             Brevo (Email)
                                                      Atributos + listas

┌──────────────────────────────────────────────────────────────────────────────────┐
│  Panel Next.js (Vercel)                                                          │
│  NextAuth + RBAC → Proxy /api/farmer/[...path] → Backend con FARMER_ADMIN_KEY    │
│  La clave de administración nunca llega al navegador.                            │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Stack Tecnológico

- **Backend:** Python 3.11+, FastAPI, `databases` + `asyncpg`, `httpx` · Render
- **Base de datos:** PostgreSQL
- **Frontend:** Next.js, NextAuth, Tailwind CSS · Vercel
- **Integraciones Comerciales:** CRM / ERP API & Tiendanube API
- **Mensajería:** WhatsApp Cloud API (Meta Graph API directo, sin intermediarios)
- **Email Marketing:** Brevo API
- **Captación WiFi:** Easy Auth / Portales Cautivos

---

## Estructura del repo

```text
main.py                    # App FastAPI: webhook Easy Auth, sincronización CRM, Brevo, SCV, antenas y redirector /go/
whatsapp_routes.py         # Router /whatsapp: config, plantillas, campañas, progreso, clics y cross-sell
whatsapp_provider.py       # MetaCloudProvider, MetaTemplateManager y MetaMediaUploader
bandeja.py                 # Bandeja de Mensajes y motor de clasificación por reglas
ficha.py                   # Endpoint GET /clientes/ficha (Ficha 360°)
web_push.py                # Avisos Web Push (VAPID)
db_ddl.py                  # Ejecución de DDL con lock_timeout y verificación de schema sin bloqueos
requirements.txt

pages/
  index.js                 # Inicio: últimos contactos captados + resumen de última campaña
  scv.js                   # Directorio unificado de Contactos (SCV)
  campanas.js              # Creación y monitoreo de Campañas
  whatsapp.js              # Gestión de Plantillas de Meta
  respuestas.js            # Bandeja de Mensajes multiagente
  antenas.js               # ABM de Antenas (APs por MAC)
  comercio/[slug]/         # Métricas y reportes por comercio adherido
  admin.js · login.js      # Administración y acceso
  api/auth/[...nextauth].js
  api/farmer/[...path].js  # Proxy autenticado hacia el backend
components/                # TopNav, BottomNav, UI kit, modales de campañas y Ficha 360°
lib/                       # Cliente API, adaptadores, utilidades de campañas y Web Push
public/sw.js               # Service Worker de notificaciones push
middleware.js              # Protección de rutas por sesión y rol
```

---

## Setup local

**Requisitos:** Python 3.11+, Node.js 18+, PostgreSQL.

```bash
git clone [https://github.com/tu-usuario/the-wifi-farmer.git](https://github.com/tu-usuario/the-wifi-farmer.git)
cd the-wifi-farmer

# Backend
python3 -m venv venv
source venv/bin/activate        # En Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # Completar con credenciales de entorno
uvicorn main:app --reload --port 10000

# Frontend (en otra terminal)
npm install
npm run dev                     # Disponible en http://localhost:3000
```

El schema de base de datos se verifica y actualiza automáticamente al iniciar (`ensure_schema` / `db_ddl`). Si el schema ya está al día, no ejecuta operaciones DDL.

---

## Variables de entorno (`.env.example`)

Nunca commitear valores reales al repositorio.

### Backend

| Variable | Uso |
|---|---|
| `DATABASE_URL` | Cadena de conexión a PostgreSQL |
| `CRM_BASE_URL` / `CRM_API_KEY` | Endpoint y clave de conexión con el sistema de gestión / CRM |
| `WEBHOOK_SECRET` | Secreto de validación para el header `X-Webhook-Secret` del portal cautivo |
| `ADMIN_API_KEY` | Header `X-Admin-Key`. Soporta múltiples claves separadas por coma para rotación sin corte de servicio |
| `META_ACCESS_TOKEN` | System User Token permanente de Meta Business |
| `META_PHONE_NUMBER_ID` | Identificador del número emisor en WhatsApp Cloud API |
| `META_WABA_ID` | Identificador de la cuenta de WhatsApp Business (gestión de plantillas) |
| `META_WA_APP_ID` | App ID en Meta Developers (subida de imágenes para encabezados de plantillas) |
| `META_GRAPH_VERSION` | Versión de Graph API (default `v23.0`) |
| `WHATSAPP_TIMEOUT_SECONDS` | Timeout de peticiones a Meta (default `15`) |
| `WA_PLANTILLA_SEGUIMIENTO` / `WA_PLANTILLA_SEGUIMIENTO_IDIOMA` | Plantilla homologada para re-contacto post 24 h (default `seguimiento_consulta` / `es_AR`) |
| `VAPID_PRIVATE_KEY` | Clave privada para notificaciones Web Push (si no se define, se autogenera en `wa_push_config`) |
| `METRICS_API_KEY` | Clave de acceso al endpoint de métricas de campaña |
| `BREVO_API_KEY` + `BREVO_LIST_*` | Sincronización de embudos de correo en Brevo |

### Frontend

| Variable | Uso |
|---|---|
| `NEXT_PUBLIC_API_URL` | URL base del backend |
| `FARMER_ADMIN_KEY` | Clave privada inyectada por el proxy server-side (nunca expuesta con `NEXT_PUBLIC_`) |
| `DASHBOARD_USERS` | Configuración de usuarios y asignación de roles |
| `NEXTAUTH_SECRET` / `NEXTAUTH_URL` | Configuración de sesión de NextAuth |
| `NEXT_PUBLIC_GO_BASE` | Dominio base para el acortador/redirector `/go/{codigo}` |

---

## Seguridad y roles (RBAC)

- Todas las peticiones del panel pasan por `/api/farmer/[...path]`: valida la sesión activa (`401`), verifica permisos según el rol (`403`) y reenvía la petición al backend firmada con `FARMER_ADMIN_KEY` y el identificador `X-Farmer-User`.
- Endpoints públicos acotados: `/health`, webhooks verificados por firma y el redirector `/go/`. Todo el resto requiere `X-Admin-Key`.

| Rol | Permisos |
|---|---|
| `admin` | Control total de configuración, plantillas, campañas, comercios, antenas y bajas. |
| `agente` | Atención en Bandeja de Mensajes (responder, enviar plantilla de seguimiento, cambiar estados), consulta de Ficha 360° y avisos Web Push. |
| `viewer` | Acceso de solo lectura, consulta de alcance en segmentos y activación de notificaciones. |

Auditoría por usuario registrada en `whatsapp_campaigns.creada_por`, `wa_bandeja_salientes.agente` y `wa_seguimiento.actualizado_por`.

---

## Motor de Campañas de WhatsApp

### Orígenes y Segmentación

- **Hotspots WiFi:** Leads capturados con opt-in en portales cautivos (con soporte de partición automática Mitad A / Mitad B para pruebas A/B).
- **Clientes Activos (`crosssell_clientes`):**
  - **E-commerce / Tienda:** Segmentación por cupo o crédito disponible, historial de compras y reactivación de compradores inactivos.
  - **Cross-Sell de Servicios:** Segmentación cruzada según productos o servicios contratados.

### Reglas de Exclusión Automática

Antes de cada despacho se excluyen automáticamente: registros en lista de bajas (`wa_bajas`), números con formato inválido, contactos impactados dentro de la ventana de **cadencia de 30 días**, duplicados entre fuentes y deduplicación estricta a un único mensaje por línea telefónica (`phone10`).

### Ejecución y Estados

- `POST /whatsapp/campaigns/send` con `background: true`: congela la audiencia objetivo, crea el registro de campaña, devuelve `campaign_id` e inicia el despacho asíncrono. El panel consulta `GET /whatsapp/campaigns/{id}/progreso` cada 2 s.
- **Corte temprano (Fail-fast):** Si Meta rechaza el recurso multimedia del encabezado en el primer envío, la campaña se aborta de inmediato para evitar errores masivos.
- **Modo prueba:** Envío síncrono acotado a números de test con prefijo `[PRUEBA]`.

| Estado | Significado |
|---|---|
| `enviando` | Tarea asíncrona en ejecución activa. |
| `parcial` | Lote completado alcanzando el tope operativo por tanda. |
| `finalizada` | Audiencia del segmento procesada en su totalidad. |
| `abortada` | Cancelación automática por rechazo de cabecera multimedia en Meta. |
| `error` | Excepción capturada con detalle en `error_motivo`. |
| `interrumpida` | Detectado por `/progreso` cuando el heartbeat supera los 180 s sin actividad. |

---

## Bandeja de Mensajes y Reglas Operativas

- Conversación unificada por teléfono normalizado (`phone10`), con filtros por origen (Todos / Campañas / Tienda).
- **Ventana de 24 horas de Meta:** Menos de 24 h desde el último mensaje entrante habilita respuesta en texto libre; pasadas las 24 h habilita el envío de la plantilla homologada `seguimiento_consulta` (máximo una cada 7 días por contacto).
- **Procesamiento de bajas:** La respuesta exacta `BAJA` ejecuta el opt-out automático. Las bajas implícitas se procesan al cerrar la conversación con resultado `"baja"`.
- **Clasificación automática por reglas:** Enrutamiento entrante en orden de prioridad (`baja` -> `reclamo` -> `no_interesa` -> `ya_cliente` -> `interes`).
- **Protección de base de datos en alta concurrencia:** Toda operación DDL se ejecuta con `SET LOCAL lock_timeout = 3s` y las consultas pesadas de conteo de segmentos utilizan control de concurrencia (`Semaphore(2)`), `statement_timeout` de 25 s y caché en memoria.# wifi-farmer-whatsapp-crm
Plataforma de captación omnicanal (Hotspots WiFi + E-commerce), CRM y motor de campañas por WhatsApp Cloud API (Meta Directo) con bandeja multiagente y Ficha 360°.
