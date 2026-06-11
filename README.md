# Plataforma de Métricas Instagram — Berta

Sistema completo de recolección, almacenamiento y visualización de métricas
de Instagram para Berta, agencia de marketing de Leones, Córdoba.
Reemplaza un proceso 100% manual (capturas de pantalla + IA + presentación)
con una plataforma automatizada para ~20 cuentas de clientes.

## Problema

El proceso original para generar reportes de Instagram era:
1. Entrar manualmente a cada cuenta
2. Sacar capturas de pantalla de las métricas
3. Pasarlas a una IA
4. La IA generaba una presentación
5. Enviar manualmente al cliente

**Problemas**: consumía horas por ciclo, no escalaba, sin historial,
imposible comparar períodos, dependía de capturas de pantalla inexactas.

## Solución

Plataforma compuesta por:
- **6 workflows n8n** corriendo automáticamente todos los días
- **Base de datos PostgreSQL** (Supabase self-hosted) con 15 tablas y 10+ vistas
- **Dashboard Next.js 14** deployado en Vercel con reportes públicos por cliente
- **Infraestructura self-hosted** en VPS Contabo con Docker + EasyPanel

## Arquitectura general

Meta Graph API
↓
n8n workflows (self-hosted · VPS Contabo)
↓
Supabase PostgreSQL (self-hosted · Docker)
↓
Next.js 14 Dashboard (Vercel)
↓
Reporte público /report/[token] (por cliente)



## Workflows n8n

### 1. Instagram Metrics Sync · 08:00 diario
Lee todas las cuentas activas desde la vista `v_active_instagram_sync_targets`,
para cada cuenta llama a la Graph API y trae:
- **Perfil**: `followers_count`, `follows_count`, `media_count`
- **Insights**: `reach`, `profile_views`, `website_clicks`, `total_interactions`,
  `likes`, `comments`, `shares`, `saves`, `accounts_engaged`, `follows_and_unfollows`

Hace `upsert` en `instagram_account_metrics_daily` con
`on_conflict=instagram_account_id,metric_date` para evitar duplicados.

**Problema técnico resuelto**: `follows_and_unfollows` devuelve un breakdown
anidado en lugar de un valor simple. Solución: parsear
`total_value.breakdowns[0].results` buscando `dimension_values === 'follow'`
y `'unfollow'` por separado.

### 2. Demographics Sync · 09:00 diario
Meta no permite combinar breakdowns en una sola llamada. Solución: 4 llamadas
separadas en cadena para cada cuenta:
1. `breakdown=age` → distribución etaria (13-17, 18-24, 25-34, 35-44, 45-54, 55-64, 65+)
2. `breakdown=gender` → F / M / U
3. `breakdown=city` → top ciudades
4. `breakdown=country` → top países

Resultado almacenado como JSONB en `instagram_account_demographics`.

### 3. Posts Sync · 09:30 diario
Trae los últimos 50 posts/reels por cuenta via `/media`. Para reels usa
`thumbnail_url` (nunca `media_url` que es el archivo de video).
Hace `upsert` en `instagram_media` con `on_conflict=instagram_media_id`.

**Problema técnico resuelto**: el nodo `Format Posts` usaba `.map()` que no
genera `pairedItem` correctos en n8n. Solución: loop `for...of` con
`pairedItem: { item: 0 }` explícito.

### 4. Token Renewal · 10:00 diario
Verifica tokens en `client_report_tokens`. Si alguno vence en menos de 7 días,
lo renueva automáticamente (+90 días).

### 5. Meta Token Refresh · 10:30 diario
Verifica tokens OAuth en `meta_connections`. Si alguno vence en menos de 30 días,
llama a `graph.facebook.com/oauth/access_token` con `grant_type=fb_exchange_token`
y actualiza el token en Supabase (+60 días).

### 6. Webhook on-demand
Endpoint `POST /berta-sync` que recibe un `accountId`, consulta la vista,
llama a la Graph API y hace upsert de métricas en tiempo real.
Usado desde el dashboard para sync manual por cuenta.

## Base de datos — Supabase PostgreSQL

### Tablas principales

| Tabla | Descripción |
|-------|-------------|
| `clients` | Clientes de la agencia |
| `instagram_accounts` | Cuentas de Instagram por cliente |
| `meta_connections` | Tokens OAuth de Meta por cuenta |
| `instagram_account_metrics_daily` | Métricas diarias (reach, likes, saves, engagement_rate, etc.) |
| `instagram_account_demographics` | Demografía en JSONB (edad, género, ciudades, países) |
| `instagram_media` | Posts y reels con métricas |
| `instagram_media_metrics_daily` | Métricas diarias por post |
| `client_report_tokens` | Tokens únicos para reportes públicos |
| `monthly_reports` | Historial de reportes generados |
| `alerts` | Sistema de alertas con severidad |
| `automation_runs` | Log de ejecuciones de workflows |
| `metric_goals` | Objetivos de métricas por cliente |

### Vistas principales

| Vista | Descripción |
|-------|-------------|
| `v_active_instagram_sync_targets` | Une clients + instagram_accounts + meta_connections para n8n |
| `v_agency_overview` | KPIs globales de la agencia |
| `v_client_dashboard_summary` | Resumen por cliente para el dashboard |
| `v_monthly_account_comparison` | Comparación mes a mes con growth % |
| `v_instagram_account_latest_metrics` | Última métrica por cuenta |
| `v_metric_goal_progress` | Progreso de objetivos vs actual |

## Dashboard Next.js 14

Deployado en Vercel. Páginas:

- **Overview**: KPIs de toda la agencia (seguidores totales, alcance,
  interacciones, engagement), gráficos de tendencia, selector 30/60/90 días
- **Clientes**: cards con logo real, métricas resumidas, botón "Ver detalle"
  y botón "Compartir informe"
- **Detalle por cliente**: KPIs con delta % vs período anterior, gráfico de
  alcance, gráfico de engagement, top 6 posts con miniaturas reales,
  sección de audiencia (género/edad/ciudades), tabla de métricas diarias
- **Reporte público `/report/[token]`**: página sin sidebar, accesible solo
  con token único, muestra todas las métricas del cliente

## Stack técnico

- **n8n** (self-hosted) — orquestador de workflows
- **Meta Graph API v20.0** — fuente de métricas e insights
- **Supabase** (self-hosted) — PostgreSQL + REST API automática
- **Next.js 14** — dashboard con App Router
- **Vercel** — deploy del dashboard
- **Docker + EasyPanel + VPS Contabo** — infraestructura self-hosted

## Estructura del proyecto

/
├── workflows/
│   ├── instagram-metrics-sync.json
│   ├── demographics-sync.json
│   ├── posts-sync.json
│   ├── token-renewal.json
│   ├── meta-token-refresh.json
│   └── webhook-sync.json
├── database/
│   └── schema.sql
└── README.md



## Variables de entorno necesarias

NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
META_APP_ID=
META_APP_SECRET=
N8N_WEBHOOK_URL=



> Las credenciales reales no están incluidas en este repositorio.
