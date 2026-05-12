# Combustible Condugas — Integración con Odoo

> Documentación para el implementador del módulo Odoo.
> Versión 1 · 2026-05-12
> URL viva: https://72.60.28.156.nip.io/fuel-doc.html

---

## 1. Contexto

Condugas opera con equipos que consumen **gasolina** y **ACPM**. El combustible se compra en canecas de 55 galones y se despacha en campo. Esta app provisional captura el despacho diario y calcula rendimientos mientras se construye el módulo Odoo definitivo.

Toda la app fue diseñada para **convivir con Odoo** y luego ser reemplazada sin pérdida de histórico. Cada entidad incluye un campo `odoo_*_id` nullable que el implementador llenará durante la migración.

---

## 2. Arquitectura actual (provisional)

```
[Celular del despachador]
       │
       ↓ HTTPS + Basic Auth (htpasswd-fuel, independiente)
[ nginx :443 ]
       │
       ├→ /fuel-index.html · /fuel-despacho.html · /fuel-admin.html · /fuel-dashboard.html · /fuel-doc.html
       │
       └→ /fuel/*   →  proxy_pass  →  127.0.0.1:8096
                                          │
                                          ↓
                                   [ agent-fuel-api.py ]
                                          │
                                          ↓
                               /root/siendo-apps/data/fuel-*.json
```

| Componente              | Tecnología                                  | Ubicación                                          |
|-------------------------|---------------------------------------------|----------------------------------------------------|
| Backend                 | Python 3 stdlib (`http.server`, sin Flask)  | `/root/agent-fuel-api.py`                          |
| Almacenamiento          | JSON files atómicos (rename-on-write)       | `/root/siendo-apps/data/fuel-*.json`               |
| UI                      | HTML/JS/CSS vanilla, sin build              | `fuel-*.html` en `/root/siendo-apps/`              |
| Reverse proxy + auth    | nginx + htpasswd bcrypt                     | `/etc/nginx/.htpasswd-fuel`                        |
| Servicio                | systemd `agent-fuel-api.service`            | Log: `/var/log/agent-fuel-api.log`                 |

---

## 3. Modelo de datos

### 3.1 `fuel-equipos.json`

```jsonc
{
  "id": "uuid",
  "codigo": "COMP-001",
  "nombre": "Compresor Sullair 185",
  "tipo": "compresor",        // compresor | vehiculo | planta | motobomba | otro
  "combustible": "acpm",      // gasolina | acpm
  "medicion": "horas",        // horas | odometro | dia
  "rendimiento_esperado": 1.2,
  "unidad_rendimiento": "hora/gal",
  "alerta_umbral_pct": 20,
  "activo": true,
  "odoo_equipo_id": null      // ← FK para integración Odoo
}
```

### 3.2 `fuel-personas.json`

```jsonc
{
  "id": "uuid",
  "nombre": "Juan Pérez",
  "cedula": "12345",
  "area": "Obra",
  "rol": "operador",          // operador | despachador | ambos
  "activo": true,
  "odoo_employee_id": null    // ← FK
}
```

### 3.3 `fuel-tanques.json` (canecas)

```jsonc
{
  "id": "uuid",
  "codigo": "TQ-20260512-001",
  "combustible": "acpm",
  "galones_iniciales": 55.0,
  "galones_actuales": 42.5,   // calculado
  "fecha_apertura": "2026-05-12T10:00:00-05:00",
  "fecha_cierre": null,
  "estado": "activo",         // activo | cerrado
  "notas": "Factura 1234"
}
```

### 3.4 `fuel-despachos.json`

```jsonc
{
  "id": "uuid",
  "fecha": "2026-05-12T15:30:00-05:00",
  "tanque_id": "uuid",
  "equipo_id": "uuid",
  "operador_id": "uuid",
  "despachador_id": "uuid",
  "galones": 5.0,
  "lectura": 1006,            // horas u odómetro; null si medicion=dia
  "notas": ""
}
```

### 3.5 Cálculo de rendimiento

Se calcula entre dos despachos consecutivos del mismo equipo:

| Medición   | Fórmula                                            | Unidad     | Mejor cuando…  |
|------------|----------------------------------------------------|------------|----------------|
| `horas`    | `(horas_n − horas_{n−1}) / galones_n`              | hora/gal   | más alto       |
| `odometro` | `(km_n − km_{n−1}) / galones_n`                    | km/gal     | más alto       |
| `dia`      | `galones_n / días entre n y n−1`                   | gal/día    | más bajo       |

Alerta cuando `|desviación_pct| > alerta_umbral_pct` (default 20 %).

---

## 4. API REST actual

Base: `https://<host>/fuel/`. Auth: HTTP Basic.

| Método | Ruta                                | Descripción                              |
|--------|-------------------------------------|------------------------------------------|
| GET    | `/equipos`                          | lista                                    |
| POST   | `/equipos`                          | alta                                     |
| PUT    | `/equipos/<id>`                     | update parcial                           |
| DELETE | `/equipos/<id>`                     | soft-delete (`activo=false`)             |
| GET    | `/personas`                         |                                          |
| POST   | `/personas`                         |                                          |
| PUT    | `/personas/<id>`                    |                                          |
| GET    | `/tanques?estado=activo\|cerrado\|all` |                                       |
| POST   | `/tanques`                          | abre caneca                              |
| PUT    | `/tanques/<id>/cerrar`              | cierra caneca                            |
| PUT    | `/tanques/<id>`                     | edita notas / código                     |
| GET    | `/despachos?desde=&hasta=&...`      |                                          |
| POST   | `/despachos`                        | retorna `{despacho, rendimiento, galones_restantes_tanque}` |
| DELETE | `/despachos/<id>`                   | revierte                                 |
| GET    | `/rendimientos?equipo=&desde=&hasta=` | historial calculado                    |
| GET    | `/kpis?periodo=semana\|mes`         | KPIs agregados                           |
| GET    | `/alertas`                          | rendimientos fuera de rango + canecas bajas + lecturas faltantes |
| GET    | `/health`                           | liveness                                 |

### Ejemplo POST /despachos

```bash
curl -u condugas:****** \
     -X POST https://72.60.28.156.nip.io/fuel/despachos \
     -H "Content-Type: application/json" \
     -d '{
       "tanque_id":   "uuid-caneca",
       "equipo_id":   "uuid-compresor",
       "operador_id": "uuid-juan",
       "galones":     5.0,
       "lectura":     1006
     }'
```

Respuesta:
```json
{
  "despacho": { "...": "..." },
  "rendimiento": { "valor": 1.2, "unidad": "hora/gal", "esperado": 1.2,
                   "desviacion_pct": 0.0, "alerta": false },
  "galones_restantes_tanque": 45.0
}
```

---

## 5. Mapping a Odoo

### 5.1 Equipos → `fleet.vehicle` (o `maintenance.equipment`)

| Campo provisional        | Odoo                                          | Notas                                                              |
|--------------------------|-----------------------------------------------|--------------------------------------------------------------------|
| `nombre`                 | `name`                                        |                                                                    |
| `codigo`                 | `license_plate` o custom `x_codigo_interno`   | Para equipos sin placa (compresores) usar campo custom             |
| `tipo`                   | Categoría `vehicle_type_id`                   | Crear categorías: compresor, planta, motobomba                     |
| `combustible`            | `fuel_type`                                   | Mapear `gasolina→gasoline`, `acpm→diesel`                          |
| `medicion`               | Custom `x_unidad_medicion` (Selection)        | horas/odometro/dia                                                 |
| `rendimiento_esperado`   | Custom `x_rendimiento_esperado` (Float)       |                                                                    |
| `alerta_umbral_pct`      | Custom `x_alerta_umbral_pct` (Float)          | Default 20                                                         |
| `activo`                 | `active`                                      | Standard Odoo                                                      |

> **Alternativa:** si no se usa Fleet, `maintenance.equipment` también sirve y trae horómetro nativo.

### 5.2 Personas → `hr.employee`

| Campo       | Odoo                                          | Notas                       |
|-------------|-----------------------------------------------|-----------------------------|
| `nombre`    | `name`                                        |                             |
| `cedula`    | `identification_id`                           |                             |
| `area`      | `department_id` o `job_id`                    |                             |
| `rol`       | Custom `x_fuel_rol` (Selection)               | operador/despachador/ambos  |
| `activo`    | `active`                                      |                             |

### 5.3 Canecas → modelo nuevo `condugas.fuel.tank`

```python
class FuelTank(models.Model):
    _name = 'condugas.fuel.tank'
    _description = 'Caneca de combustible'

    name              = fields.Char('Código', required=True)
    fuel_type         = fields.Selection(
                            [('gasoline','Gasolina'),('diesel','ACPM')],
                            required=True)
    gallons_initial   = fields.Float('Galones iniciales', default=55.0)
    gallons_used      = fields.Float(compute='_compute_used', store=True)
    gallons_remaining = fields.Float(compute='_compute_remaining', store=True)
    open_date         = fields.Datetime(default=fields.Datetime.now)
    close_date        = fields.Datetime()
    state             = fields.Selection(
                            [('active','Activa'),('closed','Cerrada')],
                            default='active')
    note              = fields.Text()
    dispatch_ids      = fields.One2many('condugas.fuel.dispatch','tank_id')
```

### 5.4 Despachos → modelo nuevo `condugas.fuel.dispatch`

```python
class FuelDispatch(models.Model):
    _name        = 'condugas.fuel.dispatch'
    _description = 'Despacho de combustible'
    _order       = 'dispatch_date desc'

    dispatch_date = fields.Datetime(required=True, default=fields.Datetime.now)
    tank_id       = fields.Many2one('condugas.fuel.tank', required=True,
                                    ondelete='restrict')
    vehicle_id    = fields.Many2one('fleet.vehicle', required=True)
    operator_id   = fields.Many2one('hr.employee', required=True,
                                    string='Operador')
    dispatcher_id = fields.Many2one('hr.employee', required=True,
                                    string='Despachador')
    gallons       = fields.Float(required=True)
    reading       = fields.Float(string='Lectura horómetro/odómetro')
    notes         = fields.Text()

    # Calculados
    yield_value   = fields.Float(compute='_compute_yield', store=True)
    yield_unit    = fields.Char(compute='_compute_yield', store=True)
    deviation_pct = fields.Float(compute='_compute_yield', store=True)
    alert         = fields.Boolean(compute='_compute_yield', store=True)
```

`_compute_yield` replica la lógica del backend actual: busca el despacho previo del mismo `vehicle_id` por `dispatch_date` y aplica la fórmula según `vehicle_id.x_unidad_medicion`.

---

## 6. Estrategia de migración

### Fase A — Sincronización dual (recomendada)

Mientras se construye el módulo Odoo, la app provisional sigue como fuente operativa. Job (cron o webhook) que:

1. Lee `fuel-*.json` (o consume `/fuel/despachos?desde=`).
2. Crea/actualiza en Odoo vía XML-RPC.
3. Escribe los `odoo_*_id` de vuelta en los JSON para que la app conozca el id Odoo.

```python
import xmlrpc.client
url, db, user, pwd = "https://odoo.condugas.co", "condugas", "api@condugas.co", "******"
common = xmlrpc.client.ServerProxy(f"{url}/xmlrpc/2/common")
uid    = common.authenticate(db, user, pwd, {})
models = xmlrpc.client.ServerProxy(f"{url}/xmlrpc/2/object")

new_id = models.execute_kw(db, uid, pwd, 'condugas.fuel.dispatch', 'create', [{
    'dispatch_date': '2026-05-12 14:30:00',
    'tank_id':       42,
    'vehicle_id':    17,
    'operator_id':   88,
    'dispatcher_id': 88,
    'gallons':       5.0,
    'reading':       1006,
}])
```

> En el mismo VPS ya existen scripts XML-RPC funcionando bajo `/root/.openclaw/workspace/scripts/` (test-odoo-connection.py, analisis-gerencial-odoo.py). Credenciales en `/root/.env-odoo`. Pueden reusarse como plantilla.

### Fase B — Cambio de fuente

Cuando el módulo Odoo esté validado:
1. Backup completo de `fuel-*.json`.
2. UI apunta a la API de Odoo (mismas rutas semánticas).
3. Mantener `/fuel/` read-only ~30 días para reconciliaciones.

### Fase C — Decomisionado

Apagar `agent-fuel-api.service`. Conservar JSON como respaldo frío. Redirigir `fuel-*.html` al módulo Odoo o eliminar.

---

## 7. Operación día a día

| Tarea                          | Quién                | Frecuencia          | Dónde                   |
|--------------------------------|----------------------|---------------------|-------------------------|
| Cargar/dar de baja equipos     | Admin Condugas       | Al adquirir/baja    | `/fuel-admin.html`      |
| Abrir caneca al recibir        | Bodega               | Cada compra         | Admin → Canecas         |
| Registrar despacho             | Bodega / despachador | Por evento          | `/fuel-despacho.html`   |
| Cerrar caneca al agotarse      | Bodega               | Cuando llega a 0    | Admin → Canecas         |
| Revisar alertas                | Supervisión          | Diario              | `/fuel-dashboard.html`  |
| Reporte mensual                | Gerencia             | Mensual             | Tablero → periodo Mes   |

### Validaciones backend
- Galones > 0 y ≤ saldo de caneca.
- Combustible equipo = combustible caneca.
- Lectura obligatoria si `medicion ∈ {horas, odometro}`.
- Tanque debe estar `activo`.
- Operador debe existir y estar activo.

---

## 8. Seguridad y accesos

- HTTPS obligatorio (Let's Encrypt).
- HTTP Basic Auth con `htpasswd -B` (bcrypt). Archivo dedicado `/etc/nginx/.htpasswd-fuel`, separado del htpasswd general del VPS.
- Agregar usuario: `sudo htpasswd -B /etc/nginx/.htpasswd-fuel <usuario>`
- `noindex` + `X-Robots-Tag` en headers.
- Rate limit nginx (`limit_req zone=write_zone burst=20`) sobre escrituras.
- Auditoría: cada despacho deja registrado `despachador_id` y timestamp.

**Pendiente producción:** migrar Basic Auth → login con sesión, idealmente SSO/OAuth contra el Odoo de Condugas.

---

## 9. Siguiente paso para el implementador

1. Validar mapeo con la versión real de Odoo en Condugas (módulos instalados, estructura organizacional).
2. Crear módulo `condugas_fuel`:
   - Modelos `condugas.fuel.tank` y `condugas.fuel.dispatch`.
   - Custom fields en `fleet.vehicle` y `hr.employee`.
   - Vistas list/form/kanban + reportes pivot/graph.
3. Endpoint XML-RPC o REST para que la app provisional pueda hacer push/pull (Fase A).
4. Plan de carga inicial:
   - Exportar vehículos/empleados Odoo existentes → poblar `odoo_*_id` en JSON.
   - Importar canecas y despachos desde JSON cuando los modelos custom existan.
5. Decidir: ¿se conserva la UI provisional como app móvil (más liviana) o se reemplaza por la app móvil Odoo?

---

_Documento canon: `/root/siendo-apps/FUEL-INTEGRACION-ODOO.md` · sincronizado con `fuel-doc.html`._
