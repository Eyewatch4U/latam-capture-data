# latam-capture-data

Captura automática del snapshot de un **shareable report de Meltwater** e
**ingesta directa** del JSON al **Cloudflare Worker** del Monitor LATAM, vía
**GitHub Actions + Playwright**.

> **Repo público a propósito**: los minutos de GitHub Actions son ilimitados y
> gratis en repos públicos (en privados free el tope es 2.000/min mes, y una
> corrida cada 15 min lo revienta). **Nada sensible vive acá**: URL, password,
> token de sync y endpoint van todos por *repository secrets* cifrados.

---

## La cadena

```
cron-job.org ──POST──▶ GitHub Actions (workflow_dispatch)
                              │
                              ▼
                   Playwright abre el reporte,
                   pasa el password gate,
                   ESPERA el .json.gz,
                   lo baja y DESCOMPRIME
                              │
                              ▼  POST { insightPage... }  + x-sync-token
              Cloudflare Worker  /ingest
                              │
                              ▼
                        Cloudflare KV  (latest_latam)
                              │
                              ▼
                   Dashboard  /monitor-latam
```

## Por qué `/ingest` y no `/update-url`

El Worker expone dos endpoints de entrada:

| Endpoint | Recibe | Quién baja el `.gz` |
|---|---|---|
| `/update-url` | `{url, sync:true}` | **el Worker** (Cloudflare) |
| `/ingest` | el JSON completo | **el runner** (GitHub) |

Meltwater devuelve **502 a IPs de datacenter**. Los datacenters de Cloudflare
entran en esa categoría → con `/update-url`, el `fetch()` del Worker contra
Meltwater falla y KV nunca se actualiza. **Make.com tampoco resolvía esto**: solo
reenviaba la URL, y Make también sale por datacenter.

Con `/ingest`, **Cloudflare nunca toca Meltwater**. El runner ya baja el `.gz` de
todas formas (para validar que traiga datos), así que mandar el JSON en vez de la
URL no cuesta nada extra y elimina el bloqueo.

## Validación de señal

Meltwater puede servir el `.gz` con la estructura completa pero con las
agregaciones en **cero** mientras regenera el snapshot. `compute_signal()`
recorre `tabs → rows → cards → fragments` sumando hits/totals/counts. Si da `0`,
el script **no ingesta** y sigue esperando un `.gz` nuevo — así no se pisa
`latest_latam` con un snapshot vacío.

## Espera, no reload

Cada `reload()` **reinicia** la generación del snapshot desde cero. Por eso el
script abre la página **una vez** y espera (hasta `MAX_WAIT_SECONDS`), tratando
los 502 del endpoint de snapshot como "todavía generando". Solo hace **un** reload
suave si hubo un stall largo sin actividad.

---

## Setup

### 1. Secrets

**Settings → Secrets and variables → Actions → Repository secrets**
(NO "Environment secrets": los workflows no declaran `environment:`).

| Secret | Valor |
|---|---|
| `MELTWATER_URL` | URL del endpoint de snapshot (con `snapshot_token`) |
| `MELTWATER_PASSWORD` | Password del reporte público |
| `WORKER_INGEST_URL` | `https://monitorlatam.reports-dca.workers.dev/ingest` |
| `SYNC_TOKEN` | Debe coincidir **exacto** con la var `SYNC_TOKEN` del Worker |
| `PROXY_URL` *(opcional)* | Proxy residencial. Último recurso, no es gratis |

### 2. Disparo cada 15 min (cron-job.org)

- **URL:** `https://api.github.com/repos/Eyewatch4U/latam-capture-data/actions/workflows/capture.yml/dispatches`
- **Method:** `POST`
- **Schedule:** `*/15 * * * *`
- **Headers:**
  - `Authorization: Bearer <PAT fine-grained, Actions: RW, scoped a este repo>`
  - `Accept: application/vnd.github+json`
  - `Content-Type: application/json`
- **Body:** `{"ref":"main"}`

> **HTTP 204 = disparo OK** (`workflow_dispatch` no devuelve cuerpo).

### 3. Verificar

```bash
# El Worker responde y tiene KV bindeado
curl -s https://monitorlatam.reports-dca.workers.dev/health
```

Disparar el workflow a mano (**Actions → Run workflow**) y leer el log.

---

## Tuning

| Var | Default | Qué hace |
|---|---|---|
| `MAX_WAIT_SECONDS` | `300` | Cuánto espera el `.gz` antes de rendirse |
| `POLL_INTERVAL_MS` | `5000` | Cada cuánto chequea |
| `STALL_RELOAD_SECONDS` | `120` | Tras cuánto stall hace el reload suave único |
| `JITTER_SECONDS` | `0` (hosted) / `45` (self-hosted) | Espera aleatoria al arranque |
| `HEADLESS` | `true` | `false` para debug local con ventana |

## Troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| `401 invalid sync token` | `SYNC_TOKEN` ≠ var del Worker | Alinear ambos valores |
| `502` persistente en `/snapshot` | Meltwater bloquea IP datacenter | Usar `capture-selfhosted.yml` (IP residencial) |
| `502` intermitente | Snapshot generándose | Subir `MAX_WAIT_SECONDS` |
| `403/404` en `/report/public/{id}/snapshot` | El front-end pollea un snapshot que aún no existe | Consecuencia del 502, no la causa |
| `señal=0` siempre | Snapshot regenerándose | El script ya espera solo; si persiste, revisar el reporte en Meltwater |
| Run en *Queued* eterno | Disparaste el self-hosted sin runner montado | Apuntar el cron a `capture.yml` |
| `FALTA el secret requerido` | Los cargaste como *Environment secrets* | Moverlos a *Repository secrets* |

### Debug local

```bash
export MELTWATER_URL='...' MELTWATER_PASSWORD='...'
export WORKER_INGEST_URL='https://monitorlatam.reports-dca.workers.dev/ingest'
export SYNC_TOKEN='...'
export HEADLESS=false MAX_WAIT_SECONDS=400
python3 capture_latam.py
```

## Archivos

```
capture_latam.py                     # captura v4 + ingesta directa
requirements.txt
.github/workflows/
  capture.yml                       # hosted (default, minutos gratis)
  capture-selfhosted.yml            # plan B: IP residencial, label "latam"
```
