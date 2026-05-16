# Superset SICEPLUS — Setup en nueva máquina

Guía paso a paso para reproducir la instalación local de Superset customizada para SICEPLUS.
Escrita para ser ejecutada por Claude Code.

---

## Prerrequisitos

- Docker Desktop instalado y corriendo
- PowerShell
- mkcert instalado (`winget install mkcert`)
- Repositorio clonado en `d:/FX/demos/superset_demo/superset` (o la ruta que corresponda)

---

## Paso 1 — Certificado SSL local

Superset necesita correr en HTTPS para poder embeberse en SICEPLUS (que también corre en HTTPS).
mkcert genera certificados confiados por el browser sin warnings.

```powershell
# Instalar la CA local de mkcert en el sistema (solo la primera vez por máquina)
mkcert -install

# Crear la carpeta de certs y generar el certificado para localhost
cd <ruta-al-repo>/superset
mkdir docker\nginx\certs
mkcert -key-file docker/nginx/certs/localhost.key -cert-file docker/nginx/certs/localhost.crt localhost
```

**Verificar:** deben existir `docker/nginx/certs/localhost.crt` y `docker/nginx/certs/localhost.key`.

---

## Paso 2 — Verificar archivos de configuración

Los siguientes archivos ya están en el repo con los cambios de SICEPLUS.
Solo verificar que existen y tienen el contenido correcto.

### 2a. `docker/pythonpath_dev/superset_config.py`

Debe contener al final del archivo (antes de los imports opcionales):

```python
FEATURE_FLAGS = {"ALERT_REPORTS": True, "DATASET_FOLDERS": True, "EMBEDDED_SUPERSET": True}

ENABLE_CORS = True
CORS_OPTIONS = {
    "supports_credentials": True,
    "allow_headers": ["*"],
    "resources": ["*"],
    "origins": ["https://localhost:44308"],
}

HTTP_HEADERS = {"X-Frame-Options": "ALLOWALL"}

TALISMAN_ENABLED = False

WTF_CSRF_ENABLED = False

PUBLIC_ROLE_LIKE = "Gamma"
```

> Si SICEPLUS corre en un puerto distinto a `44308`, actualizar `origins` con el puerto correcto.

### 2b. `docker/nginx/templates/superset.conf.template`

Debe tener dos bloques `server`: uno para HTTP (puerto 80) y uno para HTTPS (puerto 443).
El bloque HTTPS debe referenciar los certificados en `/etc/nginx/certs/`.

### 2c. `docker-compose.yml`

El servicio `nginx` debe tener:
```yaml
ports:
  - "${NGINX_PORT:-80}:80"
  - "${NGINX_SSL_PORT:-443}:443"
volumes:
  - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
  - ./docker/nginx/templates:/etc/nginx/templates:ro
  - ./docker/nginx/certs:/etc/nginx/certs:ro
```

### 2d. `docker/requirements-local.txt`

Debe existir con contenido:
```
pymssql
```

---

## Paso 3 — Levantar Docker

```powershell
cd <ruta-al-repo>/superset
docker compose up -d
```

La primera vez tarda 10-15 minutos porque:
- Descarga imágenes
- Compila el frontend con webpack (modo dev)
- Ejecuta migraciones de base de datos (`superset-init`)

**Monitorear el progreso:**
```powershell
# Ver si el contenedor principal está healthy
docker ps --filter "name=superset-superset-1" --format "{{.Status}}"

# Ver logs del webpack (esperar "webpack compiled successfully")
docker logs superset-superset-node-1 --follow
```

Cuando `superset-superset-1` muestre `(healthy)`, Superset está listo.

---

## Paso 4 — Instalar pymssql en el contenedor

pymssql es necesario para conectar Superset a SQL Server sin depender de drivers ODBC.

```powershell
docker exec superset-superset-1 pip install pymssql
```

---

## Paso 5 — Sincronizar permisos

Aplica `PUBLIC_ROLE_LIKE = "Gamma"` al rol Public para que el usuario guest del embedding
tenga los permisos necesarios.

```powershell
docker exec superset-superset-1 superset init
```

---

## Paso 6 — Verificar acceso

Abrir en el browser:
- Admin UI: `http://localhost:8088` → usuario `admin`, contraseña `admin`
- HTTPS (nginx): `https://localhost` → mismo login

Si `https://localhost` no abre, verificar que los certificados existen y que nginx levantó sin errores:
```powershell
docker compose logs nginx --tail 20
```

---

## Paso 7 — Configurar la conexión a SQL Server

1. En `http://localhost:8088` → **Settings → Database Connections → + Database**
2. Seleccionar **Other** o **Microsoft SQL Server**
3. En **SQLAlchemy URI** ingresar:
   ```
   mssql+pymssql://usuario:clave@servidor:puerto/basedatos
   ```
   Ejemplo para QA:
   ```
   mssql+pymssql://Sql:FxMoa2017@fxmoa01.database.windows.net/qasiceplus
   ```
4. **Test Connection** → debe decir "Connection looks good!"
5. **Save**

> Si ya existe una conexión con `mssql+pyodbc`, editarla y reemplazar el URI completo eliminando todo el `?driver=...&TDS_Version=...`.

---

## Paso 8 — Importar o recrear el dashboard

### Opción A — Importar desde archivo (recomendado)

Si existe un archivo de export del dashboard:
1. **Dashboards → Import** → subir el archivo `.zip`
2. El dashboard se importa con todos sus charts y datasets

### Opción B — Recrear manualmente

1. Crear los datasets apuntando a las tablas/vistas de SQL Server
2. Crear los charts
3. Crear el dashboard y agregar los charts

---

## Paso 9 — Habilitar embedding del dashboard

Este paso genera el UUID de embedding que usa SICEPLUS.

1. Abrir el dashboard → menú `⋮` (tres puntos, arriba a la derecha) → **Embed dashboard**
2. Click en **Enable embedding**
3. Copiar el UUID que aparece (formato: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`)

---

## Paso 10 — Actualizar el UUID en SICEPLUS

Si el UUID cambió respecto al original, actualizar en dos lugares:

### `siceplus-webUI/src/Fx.webUI/js/controladores/cInicio.js`

Buscar todas las ocurrencias de `d86b3ead-36df-41eb-b659-286b056af7f7` y reemplazar por el nuevo UUID:

```javascript
// En CargarDashboardSuperset():
const strUrl = `${this.ServidorAPI()}/Superset/ObtenerGuestToken?strDashboardId=<NUEVO-UUID>`;

// En embedDashboard():
await supersetEmbeddedSdk.embedDashboard({
    id: '<NUEVO-UUID>',
    ...
});

// En ObtenerGuestTokenAsync():
const strUrl = `${this.ServidorAPI()}/Superset/ObtenerGuestToken?strDashboardId=<NUEVO-UUID>`;
```

---

## Paso 11 — Verificar navegación por click en charts

Los charts del dashboard envían un `postMessage` al hacer click, y SICEPLUS escucha ese mensaje
para navegar a páginas específicas.

El mapeo está en `cInicio.js` en el método `RegistrarNavegacionSuperset`:

```javascript
const cNavegacion = {
    'Estados OC':      '/pantalla/OrdenCompra.aspx',
    'OC Confirmadas':  '/pantalla/LegajoImportacion.aspx',
};
```

Para agregar más charts: agregar entradas al objeto con el nombre exacto del chart en Superset
como clave y la URL de SICEPLUS como valor.

---

## Verificación final

| Qué verificar | Cómo |
|---|---|
| Superset accesible | `http://localhost:8088` carga el login |
| HTTPS funciona | `https://localhost` carga sin warnings |
| Dashboard embebido en SICEPLUS | Pantalla Inicio muestra el dashboard |
| Click en chart navega | Click en "Estados OC" → abre Orden de Compra |
| Sin errores en consola | DevTools → Console sin errores rojos |

---

## Troubleshooting

### nginx no levanta
```powershell
docker compose logs nginx --tail 30
```
Causa más común: certificados no generados. Repetir Paso 1.

### Error 403 en guest_token
```powershell
docker exec superset-superset-1 superset init
```
Repetir Paso 5.

### Error de conexión SQL Server (pyodbc / TDS_Version)
Verificar que la URI usa `mssql+pymssql://` (no `mssql+pyodbc://`).
Repetir Paso 7.

### Dashboard no se ve / "Something went wrong with embedded authentication"
1. Verificar que el dashboard tiene embedding habilitado (Paso 9)
2. Verificar que el UUID en `cInicio.js` coincide con el de Superset (Paso 10)
3. Verificar que `superset init` corrió (Paso 5)

### Puerto 443 ocupado en Windows
Agregar en `docker/.env-local`:
```
NGINX_SSL_PORT=8443
```
Y actualizar `appsettings.json` en SICEPLUS:
```json
"Superset": {
  "Url": "https://localhost:8443"
}
```

### Charts no envían postMessage al hacer click
El frontend de Superset (webpack dev) debe estar compilado con los cambios en `ChartHolder.tsx`.
Verificar que el archivo tiene el `handleChartClick` con `window.parent.postMessage`.
Si no, el webpack lo recompilará automáticamente al detectar el cambio.
