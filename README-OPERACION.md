# Operacion siendo-apps (VPS 72.60.28.156)

> Snapshot 2026-05-14 — Fase 2 consolidacion arquitectura.
> Mantener actualizado cuando cambien backends, agentes o endpoints.

## 1. Punto de entrada

- URL publica: `https://72.60.28.156.nip.io` (basic auth via `/etc/nginx/.htpasswd`)
- `/` redirige automaticamente a `/dashboard.html` (cambio Fase 2 2026-05-14)
- Health: `https://72.60.28.156.nip.io/health` (sin auth)
- App combustible: `/fuel-*.html` con `.htpasswd-fuel` (auth separada)
- Reportes financieros: `/reportes/privado/` con `.htpasswd-reportes` (auth separada — pendiente activar Fase 2)
- Vault privado: bare repo en `/root/repos/siendo-vault.git` (NO Github)
- `_local/` privado: bare repo en `/root/repos/siendo-local.git` (NO Github)

## 2. Frontend (HTML servidos por nginx)

Raiz: `/root/siendo-apps/` — espejado en `https://github.com/SerJefe/siendo-apps`.
nginx sirve desde `/var/www/dashboard/` (sync via `sync-dashboard.sh` cron diario).

| Archivo | Funcion | Consume |
|---|---|---|
| `index.html` | Redirect a /dashboard.html | — |
| `dashboard.html` | Entrada operativa principal | briefing, personas, roles, KPIs Odoo, M365, research |
| `priorizador.html` | Vista prioridades/tareas | `tareas-tracker.json` |
| `panorama.html` | Redirect a `/dashboard.html#panorama` (deprecado Fase 3) | `roles.json`, `roles-vivos.json`, `odoo-kpis.json`, `research-latest.json` |
| `briefing-hub.html` | Redirect a `/dashboard.html#briefing` (deprecado Fase 3) | `briefing-*.json`, `resumen-*.json` |
| `mapa-neuronal.html` | Mapa conceptual interactivo | `neural-activity.json` |
| `rigor.html` | Pagina movil rigor semanal | `rigor-semanal-*.json` |
| `siendo-premium.html` | Landing publica marca | — |
| `clientes/*.html` | Briefings clientes (Adolfo, Osdab, Vendemos, Doña, Eje Cafetero) | — (estaticos) |
| `reportes/*.html` | Reportes ejecutivos publicos (con basic auth global) | — (estaticos) |
| `reportes/privado/*.html` | Reportes financieros Condugas | — (auth extra .htpasswd-reportes) |
| `tools/*.html` | Familia, ejercicio, recetas | — (estaticos) |
| `fuel-*.html` (5) | App combustible Condugas | API loopback 8096 |

## 3. Backends FastAPI

| Puerto | Servicio | Codigo | Endpoint nginx | Auth |
|---|---|---|---|---|
| 8099 | task-toggle-api | `/root/task-toggle-api.py` | `/tasks/` | basic + token |
| 8098 | alertas-api | `/root/alertas-api.py` | `/alertas/` | basic + token |
| 8097 | agent-capturador | `/root/agent-capturador.py` (systemd) | `/capture/` | basic + token |
| 8096 | agent-fuel-api | `/root/agent-fuel-api.py` (systemd) | (proxy fuel-*.html) | htpasswd-fuel |

Token actual para `/tasks` `/alertas` `/capture` hardcoded en `/etc/nginx/sites-enabled/apis` — rotarlo cuando convenga.

## 4. Agentes cron (14)

```
*/15 * * * *       generate-ics.py              # ICS calendario
*/30 * * * *       agent-roles-status.py        # estado 8 roles
*/30 * * * *       enrich-personas.py           # enriquecedor personas
*/30 * * * *       agent-delegation-suggester.py # sugerencias delegacion
1,31 * * * *       enrich-roles.py              # enriquecedor roles
0 */4 * * *        agent-seguimiento.py         # seguimiento pendientes
30 10 * * *        agent-odoo-kpis-v2.py        # KPIs Odoo
0 11 * * 1-6       agent-revisor-diario.py      # revisor diario L-S
30 11 * * *        limitless-daily.sh           # pull Limitless
30 11 * * 1-6      agent-coherencia.py          # alertas coherencia
45 11 * * 1-6      agent-kpi-updater.py         # KPIs derivados
0 11 * * 1         agent-research-semanal.py    # research lunes
0 17 * * 5         agent-rigor-semanal.py       # rigor viernes 17:00
30 12 * * 0        resumen-semanal.sh           # resumen domingo + Telegram
```

Logs: `/var/log/agent-*.log`.

## 5. Servicios systemd

```
agent-capturador.service   # FastAPI puerto 8097
agent-fuel-api.service     # FastAPI puerto 8096
monarx-agent.service       # seguridad externa (no nuestro)
```

## 6. Datos (JSON snapshots)

`/root/siendo-apps/data/` ~80 archivos JSON. Categorias:

- Briefings datados: `briefing-YYYY-MM-DD.json`
- Resumenes semanales: `resumen-YYYY-Wnn.json`
- Rigor semanal: `rigor-semanal-YYYY-Wnn.json`
- Research IA: `research-YYYY-MM-DD.json`
- Snapshots de estado: `roles.json`, `roles-status.json`, `roles-vivos.json`, `personas.json`, `responsables.json`, `proyectos.json`, `kpis.json`, `odoo-kpis.json`, `tareas-tracker.json`, `tareas-pendientes.json`
- Alertas: `alertas.json`, `alertas-coherencia.json`, `alertas-seguimiento.json`
- Fuel app: `fuel-personas.json`, `fuel-equipos.json`, `fuel-despachos.json`, `fuel-tanques.json`
- M365 (escrito por PC, no VPS): `ms-mail-radar.json`, `ms-outlook-calendar.json`
- Estado UI mutable: `dismissed-suggestions.json`, `mail-dismissed.json`, `rechazos-recientes.json`
- DIA Limitless parseado: `dia-v2/YYYY-MM-DD.json`

## 7. Despliegue

Hoy: `git push origin main` desde `/root/siendo-apps/` espeja a Github. Para que cambios HTML/JS aparezcan en URL publica hay que correr `sync-dashboard.sh` que copia a `/var/www/dashboard/` (cron diario 12:00 UTC).

Recargar servicios systemd manualmente cuando edites `.py` de servicio: `systemctl restart agent-capturador`.

Sin CI/CD. Sin tests. Sin backups automaticos.

## 8. Cambios Fase 2 (2026-05-14)

- `index.html` reemplazado por redirect a `/dashboard.html`. Backup: `index.html.bak-pre-redirect-20260514`.
- `Condugas_Roadmap_Etapa3.html` y `Condugas_Habilitadores_Final.html` movidos de `/data/` (ubicacion incorrecta) a `/reportes/`.
- `roles-panorama.html` borrado (huerfano, versión anterior reemplazada por `panorama.html`).
- Directorio `/reportes/privado/` creado para 6 reportes financieros PC (pendiente subir + .htpasswd-reportes + nginx reload).

## 9. Fosiles documentados (limpieza Fase 1)

- `/root/temp_*.py` (4 archivos) — borrados 2026-05-13.
- `/root/siendo-apps/.clawhub/` — borrado 2026-05-13.
- `/root/.openclaw/` — **conservado** por decision explicita del sunset 2026-05-13.

## 10. Cambios Fase 3 (2026-05-14)

- **Rollback nav.js router**: la nav unificada inyectada en 6 HTMLs duplicaba los tabs internos del dashboard. Borrado del repo y de produccion.
- **panorama.html y briefing-hub.html deprecados**: convertidos en redirects a `/dashboard.html#panorama` y `/dashboard.html#briefing`. Backups: `*.bak-pre-redirect-20260514`.
- **Hash routing en dashboard.html**: `location.hash` activa el tab correspondiente al cargar (`#panorama`, `#briefing`, `#inbox`, `#agenda`, `#equipo`, `#personas`, `#radar`).
- **Hero-links del dashboard** actualizados: Panorama y Briefing apuntan a hash interno (no round-trip), Mapa y Rigor mantienen link a HTML standalone.

