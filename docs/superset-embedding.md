# Embeber Apache Superset en SICEPLUS

## Objetivo

Mostrar el dashboard Superset de UUID `28a13ea7-9b23-4776-bed7-c151f192ec09` en la pantalla Inicio de SICEPLUS.

---

## Estado al iniciar el trabajo

Cambios ya presentes en la rama antes de esta implementación:

| Archivo | Estado | Contenido |
|---|---|---|
| `IConfiguracionSuperset.cs` | Nuevo | Interface con `Url`, `Usuario`, `Clave` |
| `IConfiguracion.cs` | Modificado | Agrega propiedad `IConfiguracionSuperset Superset` |
| `appsettings.json` | Modificado | Agrega sección `Superset` con valores de desarrollo |

---

## Flujo de embedding (cómo funciona Superset)

```
1. Backend llama POST /api/v1/security/login  (usuario/clave desde appsettings)
   → Superset retorna { access_token }

2. Backend llama POST /api/v1/security/guest_token/  (con Bearer access_token)
   Body: { resources: [{ type: "dashboard", id: UUID }], rls: [], user: { ... } }
   → Superset retorna { token }  ← guest token de corta duración (~5 min)

3. API SICEPLUS retorna { Url, Token } al frontend

4. Frontend usa @superset-ui/embedded-sdk con el guest token
   → SDK crea un iframe apuntando a Superset con sesión autenticada
```

---

## Archivos creados / modificados

### Backend — `siceplus-api`

#### Capa Aplicación (`Fx.Aplicacion`)

**Nuevo:** `Aplicacion/Interfaces/Configuracion/IConfiguracionSuperset.cs`
```csharp
public interface IConfiguracionSuperset
{
    string Url { get; set; }
    string Usuario { get; set; }
    string Clave { get; set; }
}
```

**Nuevo:** `Servicios/Procesos/Integracion/IIntegracionSuperset.cs`
```csharp
public interface IIntegracionSuperset
{
    Task<string> ObtenerGuestTokenAsync(string strDashboardId);
}
```

**Nuevo:** `Modelos/ModelosSuperset/ConsultaTokenSuperset.cs`
```csharp
public class ConsultaTokenSuperset
{
    public string Url { get; set; }
    public string Token { get; set; }
}
```

---

#### Capa Presentación (`Fx.Api`)

**Modificado:** `Implementacion/Configuracion.cs`
- Agrega clase `ConfiguracionSuperset : IConfiguracionSuperset`
- Agrega propiedad en `Configuracion`: `public IConfiguracionSuperset Superset { get; set; } = new ConfiguracionSuperset();`

**Modificado:** `Extenciones/Configuracion.cs` — método `ConfiguraIntegraciones`
```csharp
IConfigurationSection cSuperset = cConfiguracion.GetSection("Superset");

cServicios.AddHttpClient("SupersetApi", cHttpClient =>
{
    cHttpClient.BaseAddress = new Uri(cSuperset.GetValue<string>("Url"));
});
```

**Nuevo:** `Controllers/Aplicacion/SupersetController.cs`
```csharp
[HttpGet]
public async Task<IActionResult> ObtenerGuestTokenAsync([FromQuery] string strDashboardId)
{
    IRespuestaApi<ConsultaTokenSuperset> cRespuestaApi = new RespuestaApi<ConsultaTokenSuperset>();
    cRespuestaApi.Respuesta = new ConsultaTokenSuperset
    {
        Url   = this.Configuracion.Superset.Url,
        Token = await this.IntegracionSuperset.ObtenerGuestTokenAsync(strDashboardId)
    };
    return Ok(cRespuestaApi);
}
```

**Modificado:** `Extenciones/Dependencias.cs` — región `#region Procesos`
```csharp
cServicios.AddScoped<IIntegracionSuperset, IntegracionSuperset>();
```

---

#### Capa Infraestructura (`Fx.Infraestructura`)

**Nuevos en** `Integracion/Superset/`:

| Archivo | Contenido |
|---|---|
| `LoginSuperset.cs` | Modelo para POST /api/v1/security/login |
| `TokenLoginSuperset.cs` | Respuesta con `access_token` |
| `SolicitudGuestTokenSuperset.cs` | Body para POST /api/v1/security/guest_token/ |
| `RespuestaGuestTokenSuperset.cs` | Respuesta con `token` |
| `IntegracionSuperset.cs` | Implementación: `LoginAsync()` + `ObtenerGuestTokenAsync()` |

Patrón de `IntegracionSuperset` (idéntico a `IntegracionTarifar`):
- Constructor recibe `IConfiguracion` + `IHttpClientFactory`
- `LoginAsync()` cachea el `access_token` en `this.TokenAcceso` (lazy, solo llama una vez)
- `ObtenerGuestTokenAsync()` llama Login primero, luego POST al endpoint de guest token

---

### Frontend — `siceplus-webUI`

#### `inicio.aspx`

Agrega antes del cierre `</main>`:
- Nuevo `<div class="row">` con panel completo (col-lg)
- Contenedor `<div id="vSuperset" style="min-height: 700px; width: 100%;">` donde el SDK monta el iframe

En el bloque de scripts al final:
```html
<script src="https://unpkg.com/@superset-ui/embedded-sdk"></script>
```
> **Producción:** descargar el bundle a `/js/plugins/superset/superset-embedded-sdk.js` y referenciar localmente.

#### `cInicio.js`

Se agregan dos métodos a la clase `Inicio`:

**`CargarDashboardSuperset()`** — llama al API, usa el SDK para montar el iframe:
```js
async CargarDashboardSuperset() {
    const strUrl = `${this.ServidorAPI()}/Superset/ObtenerGuestToken`
                 + `?strDashboardId=28a13ea7-9b23-4776-bed7-c151f192ec09`;

    const cHttpRespuesta = await fetch(strUrl, {
        method: 'GET',
        headers: this.CabeceraHttp(),
        signal: AbortSignal.timeout(_Config.TimeOut)
    });

    const cDatos = await cHttpRespuesta.json();

    if (cDatos.Respuesta) {
        await supersetEmbeddedSdk.embedDashboard({
            id: '28a13ea7-9b23-4776-bed7-c151f192ec09',
            supersetDomain: cDatos.Respuesta.Url,
            mountPoint: document.getElementById('vSuperset'),
            fetchGuestToken: () => this.ObtenerGuestTokenAsync(),
            dashboardUiConfig: { hideTitle: false, hideTab: true }
        });
    }
}
```

**`ObtenerGuestTokenAsync()`** — utilizado por el SDK para refrescar el token:
```js
async ObtenerGuestTokenAsync() {
    const strUrl = `${this.ServidorAPI()}/Superset/ObtenerGuestToken`
                 + `?strDashboardId=28a13ea7-9b23-4776-bed7-c151f192ec09`;

    const cHttpRespuesta = await fetch(strUrl, {
        method: 'GET',
        headers: this.CabeceraHttp(),
        signal: AbortSignal.timeout(_Config.TimeOut)
    });

    const cDatos = await cHttpRespuesta.json();

    return cDatos.Respuesta?.Token;
}
```

En `IniciarAsync()` se agrega al final: `await this.CargarDashboardSuperset();`

---

## Configuración requerida en Superset

Para que el embedding funcione, Superset necesita:

1. **Habilitar feature flag** en `superset_config.py`:
```python
FEATURE_FLAGS = {
    "EMBEDDED_SUPERSET": True
}
```

2. **Configurar CORS** para aceptar requests desde el dominio de SICEPLUS:
```python
CORS_OPTIONS = {
    "supports_credentials": True,
    "allow_headers": ["*"],
    "resources": ["*"],
    "origins": ["https://tu-dominio-siceplus.com"]
}
```

3. **Habilitar el dashboard para embedding** desde la UI de Superset:
   - Abrir el dashboard `28a13ea7-9b23-4776-bed7-c151f192ec09`
   - Menú "..." → "Embed dashboard"
   - Copiar el UUID generado (confirma que sea el mismo)

---

## Patrón de referencia

Esta implementación sigue exactamente el patrón de `IntegracionTarifar`:

| Aspecto | Tarifar | Superset |
|---|---|---|
| Interface config | `IConfiguracionTarifar` | `IConfiguracionSuperset` |
| HttpClient | `"TarifarApi"` | `"SupersetApi"` |
| Login | `LoginAsync()` con Bearer | `LoginAsync()` con Bearer |
| Modelo retorno | `CotizacionResultado` | `ConsultaTokenSuperset` |
| DI registro | `AddScoped<IIntegracionTarifar>` | `AddScoped<IIntegracionSuperset>` |
