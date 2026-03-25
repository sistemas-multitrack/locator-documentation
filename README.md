# Links de Locator en Wialon (crear y cancelar)

Este documento resume cómo se generan los enlaces de **Locator** y cómo **revocarlos** con la API `token/update` de Wialon.

## Variables de entorno relevantes

| Variable | Rol |
|----------|-----|
| `WIALON_API_URL` | Endpoint AJAX de Wialon. Por defecto: `https://hst-api.wialon.com/wialon/ajax.html`. Si tu cuenta está en otro host (por ejemplo **Europa**: `https://hst-api.wialon.eu/wialon/ajax.html`), debes alinearlo con el host donde inicias sesión. |
| `WIALON_LOCATOR_BASE_URL` | URL base de la página Locator que recibe el token en el query `t`. Por defecto: `https://rastreo.multitrack.com.mx/locator/index.html?t=` |

El enlace final que ve el usuario es:

```text
{WIALON_LOCATOR_BASE_URL}{encodeURIComponent(token_h)}
```

donde `token_h` es el valor del campo `h` devuelto por Wialon al crear el token.

---

## Cómo crear un link de Locator


### 1. Iniciar sesión con el token de usuario

**Servicio:** `token/login`  
**Parámetros:** `token` (token proporcionado por Multitrack), `fl: 1`  
**Respuesta:** objeto con `eid` → ese valor es el **`sid`** de sesión para las llamadas siguientes.

### 2. Resolver nombres de unidades a IDs

Para cada nombre de unidad, se llama **`core/search_items`** con:

- `spec.itemsType`: `"avl_unit"`
- `spec.propName`: `"sys_name"`
- `spec.propValueMask`: nombre exacto buscado
- `flags`, `from`, `to`, etc., según el código

Si alguna unidad no aparece, el flujo reporta `missingUnits` y no crea el link.

### 3. Crear el token Locator

**Servicio:** `token/update`  
**Sesión:** incluir `sid` en el cuerpo (igual que el resto de peticiones autenticadas).

**Parámetros (creación), alineados con el código:**

| Campo | Valor en código | Notas |
|-------|-----------------|--------|
| `callMode` | `"create"` | Obligatorio para crear. |
| `app` | `"locator"` | Tipo de aplicación del token. |
| `at` | `0` | En la implementación actual se envía `0`. |
| `dur` | segundos de vigencia | Ej.: días × 86400 u horas × 3600. |
| `fl` | `0x100` (256) | Flags del token; coincide con el `fl` del ejemplo de borrado. |
| `p` | `"{}"` | Cadena JSON (objeto vacío en este proyecto). |
| `items` | array de IDs numéricos | IDs de unidades obtenidos en el paso 2. |

**Respuesta:** objeto con campo **`h`**: hash/token del locator. Ese es el que se concatena a `WIALON_LOCATOR_BASE_URL` (con `encodeURIComponent`).


### Formato HTTP común a todas las llamadas

Todas van como **POST** `application/x-www-form-urlencoded` al `WIALON_API_URL` con:

- `svc`: nombre del servicio (ej. `token/login`, `core/search_items`, `token/update`, `core/logout`)
- `params`: JSON stringificado del objeto de parámetros del servicio
- `sid`: solo cuando ya hay sesión (no en el primer `token/login`)

Esto coincide con lo que hace `wialonRequest` en el mismo archivo.

---

## Cómo cancelar (eliminar) un link de Locator

Wialon expone la misma **`token/update`**, pero con **`callMode`: `"delete"`**. Debes usar una **sesión válida** (`sid` obtenido con `token/login` usando un token de usuario con permisos adecuados) y enviar los datos que identifican el token Locator que quieres borrar.

En el `curl` de referencia, el cuerpo decodificado es conceptualmente:

- `params`: JSON con al menos:
  - `callMode`: `"delete"`
  - `h`: el token/hash del locator (el mismo que recibiste al crear y que viaja en el query `t=` del link)
  - `app`: `"locator"`
  - `at`, `dur`, `fl`, `p`, `items`: deben **coincidir con los parámetros con los que se creó** el token (Wialon usa ese conjunto para identificar el registro). En el ejemplo:
    - `at`: timestamp Unix asociado al token
    - `dur`: duración en segundos (ej. `3600`)
    - `fl`: `256` (`0x100`)
    - `p`: string JSON interno, p. ej. `{"note":"","zones":0,"tracks":0}` (si creaste con `"{}"`, guarda lo que devolvió la creación o prueba el par idéntico al de creación según documentación Wialon)
    - `items`: array de IDs de unidades, p. ej. `[22732798]`
- `sid`: la sesión activa del usuario que elimina el token

### Ejemplo de `curl` (equivalente lógico al de Windows)

Ajusta `HOST`, `SID`, `TOKEN` y el JSON de `params` a tu caso (mismo host que usas para login, mismos `at`/`dur`/`p`/`items` que la creación).

```bash
curl "https://hst-api.wialon.eu/wialon/ajax.html?svc=token/update" \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode 'params={"callMode":"delete","h":"TOKEN WIALON","app":"locator",
  "at":1774464479,"dur":3600,"fl":256,
  "p":"{\"note\":\"\",\"zones\":0,\"tracks\":0}","items":[22732798]}' \
  --data-urlencode "sid={SID}"
```

En PowerShell/cmd el mismo contenido puede armarse con `^` para continuar líneas y `%` codificado en URLs; lo importante es que **`params` sea un único JSON válido** y **`sid`** sea el de una sesión abierta con `token/login`.



## Referencia rápida en código

Creación del link (resumen):

```
async function createWialonLocatorLink(args: {
  wialonToken: string;
  unitNames: string[];
  durationSeconds: number;
}): Promise<{url: string; missingUnits: string[]}> {
  const loginResponse = await wialonRequest<{eid?: string}>("token/login", {
    token: args.wialonToken,
    fl: 1,
  });

  const sid = loginResponse.eid;
  // ... búsqueda de unidades ...

    const tokenResponse = await wialonRequest<{h?: string}>(
      "token/update",
      {
        callMode: "create",
        app: "locator",
        at: 0,
        dur: args.durationSeconds,
        fl: 0x100,
        p: "{}",
        items: unitIds,
      },
      sid
    );

    const locatorToken = tokenResponse.h;
    // ...
    return {
      url: `${WIALON_LOCATOR_BASE_URL}${encodeURIComponent(locatorToken)}`,
      missingUnits: [],
    };
  // ... logout ...
}
```

---

## Resumen

| Acción | Servicio | `callMode` | Requiere `sid` |
|--------|----------|------------|----------------|
| Crear link Locator | `token/update` | `create` | Sí (tras `token/login`) |
| Cancelar link Locator | `token/update` | `delete` | Sí |
| Sesión | `token/login` / `core/logout` | — | Login sin `sid`; resto con `sid` |

El enlace público no es la URL de la API, sino **`WIALON_LOCATOR_BASE_URL` + token `h` codificado**, generado solo después de un `create` exitoso.
