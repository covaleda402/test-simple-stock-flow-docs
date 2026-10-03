# Contrato de API — Simple Stock Flow

> **Qué es este documento.** La forma exacta de cada petición y de cada respuesta de la API, con
> **todas** las decisiones de comportamiento tomadas. Sustituye a la lista de huecos abiertos que
> vive en [`architecture.md`](architecture.md) §3.1.1: aquí no queda ninguno abierto.
>
> **Para qué sirve.** Para que T-04, T-06, T-07 y T-08 se puedan implementar **sin inventarse nada**.
> Si al escribir código hay que elegir entre dos formas de responder y este documento no lo dice,
> eso es un fallo de este documento, no una libertad del implementador.
>
> **Fuente de verdad y fuentes ejecutables.** Este documento manda. La documentación OpenAPI que
> FastAPI genera sola (`/docs`, `/openapi.json`) es un subproducto y, si discrepa, **el código está
> mal**. Las formas viven en dos sitios del repositorio `simple-stock-flow-api`: los **DTO pydantic**
> del adaptador REST (`src/stockflow/adapters/inbound/api/schemas.py`) y los **resultados de los
> puertos de entrada**, dataclasses de `src/stockflow/application/ports/inbound/`. El único
> consumidor es el DTO TypeScript de la app (`src/infrastructure/http/dto/api.dto.ts`, repositorio
> `simple-stock-flow-app`). Los tres deben decir lo mismo que las fichas de §4.
>
> **Estado.** Especificación previa a la implementación: nada de lo descrito existe todavía. Cada
> afirmación lleva implícita la comprobación que la demostrará; las sondas de conformidad previstas
> están en el [anexo A](#anexo-a--sondas-de-conformidad-previstas).

---

## 1. Convenciones comunes

Todo lo de esta sección aplica a **todos** los endpoints y no se repite en cada ficha.

| Aspecto | Valor |
|---|---|
| **Base directa** | `http://localhost:8000` — el puerto interno del servicio `api`. En producción no se publica; solo el override `docker-compose.dev.yml` (perfil dev) lo publica al host, o se usa al ejecutar Uvicorn en local |
| **Base a través de la app** | `http://localhost:8080` — nginx proxea `/api/` y `/media/` hacia `api:8000` **sin quitar el prefijo**. `/health` **no** se proxea. Requisitos del `nginx.conf` de la app: [E-15](#e-15--get-mediakey) y [E-08](#e-08--post-apiproductsidimage) |
| **Formato** | JSON, `UTF-8` |
| **Nombres de campo** | **camelCase en el cable**, en petición y respuesta. Los DTO pydantic declaran el nombre Python en `snake_case` y el alias camelCase con `alias_generator=to_camel` y `populate_by_name=True`; las respuestas se serializan **por alias** (`response_model_by_alias=True`, que es el valor por omisión). Un campo cuyo nombre JSON es una palabra reservada de Python (`from`) declara el alias a mano |
| **Autenticación** | `Authorization: Bearer <jwt>`. El token lo emite `POST /api/auth/login` y vive **60 minutos** (configuración del servicio, no secreta; por omisión 60). Toda ruta exige token **salvo** `POST /api/auth/login`, `GET /health` y `GET /media/{key}`: la protección es **por omisión** (dependencia a nivel de router) y la excepción se declara, nunca al revés. Un test recorre las rutas registradas y comprueba que ninguna ruta fuera de esas tres responde algo distinto de 401 sin token |
| **Reloj admitido** | `leeway` de **30 segundos** al validar `exp` con PyJWT |
| **Identificadores** | `uuid` en texto, forma canónica con guiones, en minúsculas (`CHAR(36)` en la base) |
| **Importes** | JSON `number`, **cuantizado a dos decimales**, redondeo *half away from zero* (`decimal.ROUND_HALF_UP` sobre `Decimal`, en el value object `Money`). Pydantic serializa `Decimal` como **cadena** por omisión: los DTO lo **sobrescriben** con un serializador de campo que emite número, porque el contrato decide `number` y la app lo consume como tal. Un importe cabe en `DECIMAL(12,2)`: los valores representables en ese rango viajan sin pérdida como número JSON. «Dos decimales» es la precisión del valor, no el texto: `32400.5` y `32400` son válidos. En la petición, un importe con más de dos decimales se **redondea** igual; con más de diez dígitos enteros es **400** |
| **Moneda** | Siempre `"COP"`. **Nunca `null`, nunca cadena vacía** — ver [D-C10](#d-c10--currency-nunca-viaja-nulo) |
| **Fechas de respuesta** | ISO 8601 con desplazamiento explícito **`+00:00`**, siempre UTC, tal como las emite `datetime.isoformat()`: `2026-09-19T22:34:36.626645+00:00` (la fracción, de hasta 6 dígitos, solo aparece si no es cero). Pydantic emite `Z` por omisión: los DTO lo **sobrescriben** con un serializador de campo. En la base viven como `DATETIME(6)` en UTC |
| **Nulos** | Un campo declarado nulo viaja como `null`, **no se omite**. Prohibido `response_model_exclude_none`, `exclude_unset` y equivalentes: borrarían `imageUrl: null` |
| **CORS** | Solo los orígenes configurados en el servicio. En el compose la app no cruza origen: nginx proxea |

### 1.1 Roles

Dos, y solo dos: **`admin`** y **`seller`**. Cualquier otro valor lo rechaza el dominio. "Autenticado"
en las fichas significa *cualquiera de los dos*.

### 1.2 Paginación

Las dos colecciones paginadas —productos y ventas— comparten petición y respuesta.

**Parámetros de consulta:**

| Parámetro | Tipo | Obligatorio | Por omisión | Límites |
|---|---|---|---|---|
| `page` | entero | no | `1` | Menor que 1 → **se sirve 1** |
| `size` | entero | no | `20` | Ausente o menor que 1 → **20**. Mayor que 100 → **100** ([D-C5](#d-c5--un-size-por-encima-del-máximo-se-recorta-al-máximo)) |

Un `page` o un `size` que **no sean enteros** son un error de enlace: **400** con la forma de
[§2.2](#22-el-400--errors-y-detail). Los parámetros se declaran como `int` **sin** `ge` ni `le`:
cualquier restricción de rango en el esquema los convertiría en 400, y el recorte es una regla de
la aplicación (`PageRequest`), no del adaptador. Ejemplo: `?size=abc` → 400.

**Respuesta — `PagedResult<T>`:**

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `items` | `T[]` | no | La página. **Vacía es `[]`**, nunca `null` |
| `page` | `number` | no | La página **servida**, ya recortada |
| `size` | `number` | no | El tamaño **servido**, ya recortado — no el pedido |
| `total` | `number` | no | Total de elementos que casan con el filtro, **no** de la página |
| `totalPages` | `number` | no | `ceil(total / size)`; `0` si `size` es `0` — ver [D-C11](#d-c11--totalpages-es-la-parte-más-frágil-del-contrato) |

### 1.3 Rangos de fecha

Los usan `GET /api/sales` y `GET /api/reports/sales`, con las mismas reglas.

| Parámetro | Tipo | Obligatorio | Por omisión | Formato aceptado |
|---|---|---|---|---|
| `from` | instante | **sí** ([D-C4](#d-c4--from-y-to-son-obligatorios-de-verdad)) | — | **ISO 8601 con desplazamiento explícito** ([D-C3](#d-c3--solo-iso-8601-con-desplazamiento-explícito)) |
| `to` | instante | **sí** | — | igual |

- **Inclusividad: `from <= sold_at < to`.** El extremo inicial entra, el final **no**
  ([D-C2](#d-c2--from-inclusivo-to-exclusivo)).
- `to` anterior a `from` → **422**, `detail` = `"La fecha final no puede ser anterior a la inicial."`
  (value object `DateRange`).
- `from == to` es un rango **vacío válido**, no un error: `DateRange` solo rechaza `to < from`.
- En una URL, el `+` de un desplazamiento (`+02:00`) debe ir codificado como `%2B`; sin codificar
  llega como espacio y es **400**. `Z` no tiene ese problema.

---

## 2. Las tres formas del cuerpo de error

FastAPI trae sus propios cuerpos de error y **ninguno coincide con el contrato**: la validación de
forma responde **422** con `{"detail": [{"loc": …, "msg": …, "type": …}]}`, y los errores que lanza
Starlette (ruta inexistente, método no permitido, seguridad) responden `{"detail": "Not Found"}`.
Por eso el adaptador REST registra **manejadores de excepción propios** —en un único módulo, que
es el **único** sitio donde una excepción se vuelve un código— y las tres formas de abajo son las
**únicas** que pueden salir. La decisión de qué lleva cada una está en
[D-C9](#d-c9--el-400-debe-llevar-detail).

**Reparto de responsabilidades entre 400 y 422** (regla de diseño, no de estilo): el **esquema
pydantic valida solo la forma** —presencia, tipo, formato—; **toda invariante de negocio** (precio
mayor que cero, `lines` no vacío, cantidad mayor que cero, nombre no vacío) la valida el dominio
y sale como **422**. En los DTO está prohibido `gt`, `ge`, `min_length` y similares sobre esos
campos: moverían un 422 a 400. El 422 por omisión de FastAPI **no debe aparecer nunca**; un test
lo vigila.

| Excepción que llega al manejador | Respuesta |
|---|---|
| `RequestValidationError` (forma de cuerpo, consulta o formulario) | **400**, §2.2 |
| `RequestValidationError` donde **todos** los errores son de la ruta (`{id}` que no es `uuid`) | **404 vacío**, §2.3 ([D-C7](#d-c7--un-id-mal-formado-es-404-y-se-documenta-como-tal)) |
| Violación de regla de negocio del dominio | **422**, §2.1 |
| `ConcurrencyConflict` de la aplicación, agotados los reintentos | **409**, §2.1 |
| Recurso inexistente / acceso denegado / no autenticado | **404 / 403 / 401 vacíos**, §2.3 |
| `StarletteHTTPException` (404 de ruta, 405, seguridad, `StaticFiles`) | Mismo código, **cuerpo vacío**, §2.3, conservando `exc.headers` |
| Cualquier otra excepción | **500**, §2.1 |

### 2.1 El 422, el 409 y el 500 — `problem+json` con `detail`

```json
{"title":"Regla de negocio violada","status":422,"detail":"La venta debe tener al menos un ítem."}
```

```json
{"title":"Conflicto con otra operación simultánea","status":409,"detail":"Otra operación modificó los datos al mismo tiempo. Inténtalo de nuevo."}
```

```json
{"title":"Error interno","status":500,"detail":"Ocurrió un error inesperado."}
```

`Content-Type: application/problem+json; charset=utf-8`. **`detail` lleva el mensaje de dominio**,
en español, y es lo que la persona usuaria acaba leyendo. El 500 es genérico: **no filtra** el
tipo de la excepción, la traza ni el SQL (se registran en el log, no en la respuesta). Ningún
endpoint debería producirlo; existe para que, si ocurre, el cuerpo siga siendo del contrato.

### 2.2 El 400 — `errors` y `detail`

Lo emite el manejador de `RequestValidationError`. El JSON roto o con tipo incorrecto, un campo
ausente, un `uuid` o una fecha mal formados de la consulta caen aquí.

```json
{"title":"Datos de entrada no válidos",
 "status":400,
 "detail":"Datos de entrada no válidos: password.",
 "errors":{"password":["Field required"]}}
```

- `application/problem+json`, `detail` **siempre presente**, en español: *«Datos de entrada no
  válidos: <campo>[, <campo>…].»*, con los nombres en camelCase del cable.
- `errors` es un objeto `{ "<campo>": ["<mensaje>", …] }`. La clave es la ruta del campo **sin el
  origen** (`body`, `query`) y en camelCase, con los índices de arreglo y los niveles unidos por
  punto: `password`, `from`, `lines.0.quantity`. Los **mensajes son los del marco, en inglés, y no
  son contrato**: sirven para depurar, no para mostrarse.
- **No lleva** `type`, `traceId`, ni `input`/`ctx`/`url` de los errores de pydantic: ni el valor
  enviado ni el nombre de ningún tipo interno salen en la respuesta.
- Un cuerpo que no es JSON válido es también **400** con esta forma (`errors` con clave `body`).

### 2.3 El 401, el 403, el 404 y el 405 — sin cuerpo

`Content-Length: 0`. No hay JSON que leer: se consigue con un manejador de `StarletteHTTPException`
que devuelve una `Response` vacía con el mismo código y las **cabeceras** de la excepción
(`WWW-Authenticate`, `Allow`). El 401 lleva `WWW-Authenticate: Bearer`, y
`Bearer error="invalid_token"` cuando el token existe pero no vale. Esto exige que la dependencia
de seguridad lance **ella misma** el 401 con esas cabeceras: no se confía en el código por omisión
de `HTTPBearer`, que según la versión de FastAPI responde 403 y `{"detail": "Not authenticated"}`.

> **Por qué la app no se rompe con esto.** El interceptor de errores de la app (en
> `src/infrastructure/interceptors/`) enumera 0, 401, 403, 404, 409 y 422 con un texto propio; para
> **todo lo demás** lee `detail`. Por eso el 400 **debe** llevar `detail` (§2.2) y los cuatro
> códigos de esta sección pueden ir vacíos.

## 3. Las once decisiones, cerradas

Las ocho que `architecture.md` §3.1.1 identificaba como **C-1 … C-8**, más tres que el contrato
necesita. Ninguna vuelve a abrirse aquí. Las que dependen del marco (D-C3, D-C7, D-C9, D-C11) incluyen cómo
se consigue con FastAPI y pydantic; **la decisión de fondo no depende del marco**.

| # | Origen | Decisión |
|---|---|---|
| [D-C1](#d-c1--get-apicategories-existe-devuelve-un-array-plano-ordenado-por-nombre) | C-1 · ¿existe el listado de categorías? | **200 con un array plano, ordenado por nombre, vacío es `[]`** |
| [D-C2](#d-c2--from-inclusivo-to-exclusivo) | C-2 · inclusividad del rango | **`from` inclusivo, `to` exclusivo** |
| [D-C3](#d-c3--solo-iso-8601-con-desplazamiento-explícito) | C-3 · formato de fecha | **ISO 8601 con desplazamiento obligatorio** |
| [D-C4](#d-c4--from-y-to-son-obligatorios-de-verdad) | C-4 · ausencia del rango | **Obligatorios de verdad: la ausencia es 400** |
| [D-C5](#d-c5--un-size-por-encima-del-máximo-se-recorta-al-máximo) | C-5 · `size` grande | **Se recorta a 100 y la respuesta devuelve `size: 100`** |
| [D-C6](#d-c6--register-deja-de-anunciar-location) | C-6 · `Location` de `register` | **No se emite la cabecera `Location`** |
| [D-C7](#d-c7--un-id-mal-formado-es-404-y-se-documenta-como-tal) | C-7 · `{id}` mal formado | **404 vacío, igual que un id inexistente** |
| [D-C8](#d-c8--el-403-se-mantiene-y-se-exige-la-prueba-que-lo-demuestre) | C-8 · el 403 | **403 vacío, distinguible del 401, y se exige la prueba** |
| [D-C9](#d-c9--el-400-debe-llevar-detail) | — | **El 400 lleva `detail` además de `errors`** |
| [D-C10](#d-c10--currency-nunca-viaja-nulo) | — | **`currency` siempre `"COP"`, también en el reporte vacío** |
| [D-C11](#d-c11--totalpages-es-la-parte-más-frágil-del-contrato) | — | **`totalPages` es obligatorio y necesita un test de contrato** |

---

### D-C1 · `GET /api/categories` existe, devuelve un array plano ordenado por nombre

**Decisión.** `200 OK` con un **array JSON plano** —sin envoltorio de paginación— de objetos
`{id, name}`, **ordenado por `name` ascendente** con la intercalación de la base. Una lista vacía
es **`200 []`**, nunca 404 y nunca 204.

**Por qué.** El orden es lo natural de la consulta del repositorio de categorías
(`ORDER BY name`) y la app lo pinta en un desplegable donde se ve: declararlo lo convierte de
accidente en obligación verificable.

**Consecuencias.**

- La intercalación de la base es **`utf8mb4_0900_ai_ci`** (MySQL 8.4): insensible a mayúsculas **y a
  acentos**. *Fontanería* se ordena como *Fontaneria*: va después de *Electricidad* y antes de
  *General*. Orden esperado con las cinco categorías de referencia: `Electricidad`, `Fontanería`,
  `General`, `Herramientas`, `Pinturas`. Cambiar la intercalación de la base **cambiaría el
  contrato**, y también la unicidad y las búsquedas (ver [E-02](#e-02--post-apiauthregister) y
  [E-03](#e-03--get-apiproducts)).
- **204 queda prohibido** aunque no haya categorías: la app hace `dtos.map(toCategory)` sobre el
  cuerpo, y un 204 sin cuerpo lo rompe.
- Sigue sin haber `POST`, `PUT` ni `DELETE` de categorías: [`architecture.md`](architecture.md) §3.1
  explica por qué, y D-10 lo cierra.

---

### D-C2 · `from` inclusivo, `to` exclusivo

**Decisión.** Una venta entra en el rango si **`from <= sold_at < to`**. Una venta cuyo `sold_at`
caiga **exactamente en `to` no entra**.

**Por qué.** Es la única de las dos opciones bajo la cual **dos reportes contiguos suman el total
del período**: `[a,b)` y `[b,c)` cubren `[a,c)` sin solaparse. Con `to` inclusivo, una venta en el
instante `b` se contaría en los dos y la suma de los parciales no cuadraría con el total.

**Consecuencias, que hay que escribir porque muerden.**

- `?from=2026-01-01T00:00:00Z&to=2026-01-31T00:00:00Z` **no incluye el 31 de enero**. Para «todo
  enero» se pide `to=2026-02-01T00:00:00Z`.
- **La app debe ajustar el extremo final.** Si la persona elige el 31 de enero en el selector, la
  app **no** puede mandar `range.to.toISOString()` tal cual: perdería todo el día 31. Debe convertir
  el día elegido en el primer instante del siguiente, y devolver el extremo inclusivo solo para
  mostrarlo. Es responsabilidad de la app, no del backend: [hueco H-3](#5-huecos-declarados-con-su-dueño).
- El extremo inicial **sí** entra: una venta exactamente en `from` cuenta.
- La comparación se hace en UTC contra `sold_at` (`DATETIME(6)`): el adaptador convierte `from` y
  `to` a UTC antes de consultar.

---

### D-C3 · Solo ISO 8601 con desplazamiento explícito

**Decisión.** `from` y `to` se aceptan **únicamente** con desplazamiento explícito: `Z` o `±HH:MM`.
Ejemplo válido: `2026-01-01T00:00:00Z`. Todo lo demás —`2026-01-01`, `2026-06-01T10:30`,
`01/06/2026`, un epoch— es **400** con la forma de §2.2 y `errors.from` / `errors.to`.

**Por qué.** Sin acotar, una fecha sin desplazamiento se interpreta con la zona del contenedor: el
mismo rango daría dos reportes distintos según dónde corra el proceso, y el reporte dejaría de ser
estable. Era una bomba de relojería que dependía de que nadie definiera `TZ`.

**Cómo se implementa sin que el marco lo estropee.** El tipo `datetime` de la consulta de FastAPI
**no sirve tal cual**: acepta `2026-01-01` (lo toma como medianoche) y un número como epoch, y con
`AwareDatetime` el epoch sigue colándose. Los parámetros llegan como `str` y se validan con un
patrón estricto —`AAAA-MM-DDThh:mm:ss[.fracción](Z|±hh:mm)`— antes de convertirlos; lo que no case
es un error de enlace (400). **Cómo se verifica:** los cuatro casos de la lista de arriba y el
rango ausente responden 400 nombrando el campo; con `Z` explícita, 200.

---

### D-C4 · `from` y `to` son obligatorios de verdad

**Decisión.** **Obligatorios.** Su ausencia es **400** con la forma de §2.2. No hay rango por
omisión.

**Por qué.** No existe ningún rango por omisión defendible —¿el último mes?, ¿todo?— y cualquiera
que se eligiese devolvería un 200 que el consumidor interpretaría como *«no hubo ventas»*. Un
reporte vacío por descuido es peor que un error. La trampa a evitar: declarar `from` y `to` con un
valor por omisión (por ejemplo `datetime.min`) para «que no falle el enlace»; omitir los dos
parámetros devolvería **200 con una página vacía**, o un 422 que acusa a la fecha final de estar
mal cuando lo que pasa es que **no se envió**.

| Petición | Respuesta |
|---|---|
| `GET /api/reports/sales` | **400**, `errors.from` y `errors.to` |
| `GET /api/reports/sales?from=2026-01-01T00:00:00Z` | **400**, `errors.to` |
| `GET /api/reports/sales?from=&to=` | **400**, `errors.from` y `errors.to` |

---

### D-C5 · Un `size` por encima del máximo se recorta al máximo

**Decisión.** Ya la tomó **CA-01.5** —*«el sistema aplica el máximo en lugar de rechazar la
petición»*— y el criterio gana. `size > 100` se sirve como **100**, y **la respuesta devuelve
`size: 100`**, no el valor pedido. `size` ausente o menor que 1 → **20**. `page` menor que 1 → **1**.

**Por qué el `size` de la respuesta es lo importante.** La app pagina con el `size` que recibe y
con `totalPages`: devolver el `size` pedido mientras se sirven 100 elementos descuadraría el número
de páginas y el paginador saltaría filas.

**Trampa a evitar.** Tratar *«demasiado grande»* y *«ausente»* como el mismo caso (`size > 100 → 20`)
es el error natural: son dos reglas distintas. Se prueba con `?size=999` → `"size": 100` y
`?size=0` → `"size": 20`.

| `size` pedido | `size` servido |
|---|---|
| ausente | 20 |
| `0` o negativo | 20 |
| `1` … `100` | el pedido |
| `101` o más | **100** |

---

### D-C6 · `register` deja de anunciar `Location`

**Decisión.** `POST /api/auth/register` responde **`201 Created` con `{"id": "<uuid>"}` y sin
cabecera `Location`**.

**Por qué.** La alternativa era exponer `GET /api/users/{id}`, y no se va a exponer: el enunciado no
pide consultar usuarios y **DP-02** ya evita construir lecturas que crucen datos personales del
operador. Anunciar una ubicación que devuelve 404 es peor que no anunciar ninguna: un cliente que
siga la cabecera falla, y falla lejos de la causa.

**Cómo se verifica.** Un test cuyo nombre lo dice entero —*register con token de administrador
crea el usuario sin anunciar `Location`*— afirma que la respuesta 201 no trae esa cabecera.

---

### D-C7 · Un `{id}` mal formado es 404, y se documenta como tal

**Decisión.** Un identificador de ruta que no es un `uuid` devuelve **404 con el cuerpo vacío**,
igual que uno que no existe.

**Por qué.** *«No es un identificador»* y *«no existe»* son indistinguibles desde fuera. Es lo
habitual, no filtra información, y distinguirlas obligaría a un tratamiento aparte en cada acción
— más código para menos seguridad.

**Cómo se consigue en FastAPI.** Si el parámetro de ruta se declara `UUID`, un valor inválido es un
`RequestValidationError` y saldría **400**. El manejador de §2 lo reencamina: cuando **todos** los
errores de la validación son de ruta (`loc[0] == "path"`), responde **404 vacío**. Un `{id}` mal
formado combinado con un cuerpo inválido sigue siendo 400. Cubre `GET`, `PUT`, `DELETE` de
productos, la subida de imagen y `GET /api/sales/{id}`.

**Consecuencia.** La app muestra *«El recurso no existe.»* también ante un identificador con una
letra de más. Es aceptable y queda escrito para que nadie lo persiga como un fallo.

---

### D-C8 · El 403 se mantiene, y se exige la prueba que lo demuestre

**Decisión.** Un token de **`seller`** en una operación de **`admin`** responde **403 con el cuerpo
vacío**, distinguible del 401 (CA-07.4). La forma no cambia.

**Y se exige la prueba.** Un test de integración con un token de `seller` real afirma **403 con
`Content-Length: 0`** en las cinco operaciones de administrador —crear, reemplazar y dar de baja un
producto, subir su imagen, y dar de alta un usuario—, y que el **mismo token** obtiene **200** en las
cuatro lecturas y llega hasta la regla de negocio en `POST /api/sales`. Nombre sugerido:
*un vendedor puede leer el catálogo y no puede escribirlo*. El usuario `seller` no lo siembra la
migración: lo crea el propio test (y la verificación de extremo a extremo) usando `register`.
Su dueño está en [H-1](#5-huecos-declarados-con-su-dueño).

---

### D-C9 · El 400 debe llevar `detail`

**Decisión.** El cuerpo canónico del error es `application/problem+json` **con `detail` siempre
presente**. El 400 lleva **`detail` además de `errors`**: `errors` identifica el campo, y `detail`
resume en una frase **en español** qué campo falló (forma exacta en §2.2). El 401, el 403, el 404 y
el 405 **van sin cuerpo**.

**Por qué así y no al revés.** El interceptor de la app ya lee `detail` para todo lo no enumerado:
ponerlo en el backend arregla el mensaje **sin tocar el front**. La alternativa —enseñar al
interceptor a leer `errors`— deja el problema para cualquier otro consumidor futuro y obliga a
traducir textos en inglés dentro de la capa de presentación.

**Por qué el 401, el 403, el 404 y el 405 van vacíos.** Los emiten la seguridad y el enrutador,
antes de llegar a ningún caso de uso; darles cuerpo obligaría a envolverlos sin ninguna ganancia,
porque el interceptor ya los enumera y les pone su propio texto. FastAPI **no** los deja vacíos
por sí solo (`{"detail": ...}`): hace falta el manejador de §2.3.

**Qué hay que escribir.** Un manejador propio de `RequestValidationError` que construya el cuerpo
de §2.2 a partir de `exc.errors()` **descartando** `input`, `ctx` y `url`. Es el [hueco H-2](#5-huecos-declarados-con-su-dueño).

---

### D-C10 · `currency` nunca viaja nulo

**Decisión.** `currency` vale **siempre `"COP"`**, en `ProductView`, en `SaleView` y en
`SalesReport` — **incluido el reporte de un rango sin ventas**. Nunca `null`, nunca `""`.

**Por qué.** La app hace `Money.of(dto.grandTotal, dto.currency)` y `Money.of` ejecuta
`currency.toUpperCase()` **sin guarda**: un `null` lanza `TypeError` y **revienta la pantalla del
reporte**. **CA-06.2** exige que un rango sin ventas devuelva un reporte vacío *y no un error*, así
que un `currency` nulo incumpliría el criterio por la puerta de atrás, en el cliente.

**De dónde sale el valor.** De la constante de moneda por omisión del dominio
(`Money.DEFAULT_CURRENCY`). **No es una columna**: el sistema es monomoneda por construcción (D-05)
y la moneda no se persiste ni se acepta en ninguna petición. El DTO la declara como `str` sin
`Optional` y sin valor por omisión `None`, y el resultado vacío del puerto de reporte **la lleva
rellena** (no depende de que haya filas de las que copiarla).

**Forma obligatoria del reporte vacío:**

```json
{"from":"2026-01-01T00:00:00+00:00","to":"2026-02-01T00:00:00+00:00",
 "salesCount":0,"grandTotal":0,"currency":"COP","rows":[]}
```

`rows` es `[]`, **nunca `null`**. `grandTotal` es `0`, no `null`. Lo mismo vale para `items` en
`PagedResult` y en `SaleView`.

---

### D-C11 · `totalPages` es la parte más frágil del contrato

**Qué es.** Un valor **derivado** (`total` y `size`), no almacenado:

```python
total_pages = 0 if size == 0 else -(-total // size)   # ceil(total / size), con enteros
```

**Decisión.** `totalPages` es **obligatorio** en toda respuesta paginada. La app lo exige
(`PagedResultDto.totalPages`, sin `?`) y su repositorio HTTP de productos lo copia al modelo de
dominio.

**Por qué hay que decirlo en voz alta.** Es un dato calculado que viaja sin que nadie lo pida: si
se modela como `@property` de un dataclass y el DTO no lo declara como campo, **no se serializa**;
si el DTO lo declara, cualquiera de estas cosas lo borraría del JSON:

1. quitar el campo del DTO (o declararlo con `exclude=True`),
2. `response_model_exclude` / `response_model_exclude_unset` en la ruta, o un `model_dump(exclude=…)`,
3. devolver desde la ruta un tipo propio que no lo tenga.

**Ninguna de las tres rompe `mypy`, el arranque ni un test que compare objetos.** La app recibiría
`undefined`, el paginador dejaría de pintar páginas y nadie sabría por qué.

**Lo que el contrato exige.** Un **test de contrato** que afirme la presencia del campo en el JSON
**serializado por la ruta real** (cuerpo de la respuesta HTTP), no en el objeto. Va en
[H-4](#5-huecos-declarados-con-su-dueño).

---

## 4. Fichas de endpoint

Quince endpoints. **Ninguna pregunta sin respuesta.**

Regla transversal que ahorra repetirla en cinco fichas: **las operaciones por identificador —`GET`,
`PUT`, `DELETE` y la subida de imagen— no distinguen un producto retirado de uno activo**. Solo el
listado del catálogo y la carga previa a vender filtran las bajas. Es el contrato de repositorios de
[ADR-003](adr/adr-003-baja-logica.md), y es lo que permite que una línea de venta histórica resuelva
su producto (CA-02.5).

---

### E-01 · `POST /api/auth/login`

| | |
|---|---|
| **Autorización** | **Anónimo** |
| **Cuerpo** | `application/json` |

**Petición**

| Campo | Tipo | Obligatorio | Notas |
|---|---|---|---|
| `username` | `string` | **sí** | Se **normaliza**: recorte de espacios y minúsculas (`User.normalize_username`). `"  ADMIN  "` inicia sesión igual que `"admin"`. El administrador de arranque tiene como `username` el valor de `ADMIN_EMAIL`, normalizado |
| `password` | `string` | **sí** | Nunca se registra ni se almacena en claro (CA-07.5) |

*Obligatorio* significa **no nulo**. Una cadena **vacía** pasa la validación de forma y muere en
la regla de negocio: `""` → 422, `null` o ausente → 400.

**200 OK — `AuthResult`**

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `accessToken` | `string` | no | JWT firmado con HS256 (PyJWT, clave `JWT_SIGNING_KEY`). Claims: `sub` (id), `unique_name` (usuario), `role`, `jti`, `exp` |
| `expiresAt` | `string` | no | Instante de vencimiento, ISO 8601 con desplazamiento. `ahora + 60 min` |
| `username` | `string` | no | El nombre **normalizado**, no el enviado |
| `role` | `string` | no | `"admin"` o `"seller"` |

**Nunca incluye el hash de la clave** (CA-07.1).

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta `username` o `password`, o el JSON no casa | §2.2, `errors.username` / `errors.password` |
| **422** | Credenciales inválidas | §2.1, `detail` = `"Usuario o contraseña incorrectos."` — **el mismo mensaje** tanto si el usuario no existe como si la clave falla (CA-07.2) |
| **405** | `GET` sobre esta ruta | vacío, con `Allow: POST` |

---

### E-02 · `POST /api/auth/register`

| | |
|---|---|
| **Autorización** | **`admin`** |

**Solo se dan de alta vendedores** — decisión **DP-04**. El rol `admin` no se crea desde aquí: lo
provisiona el despliegue al arrancar, desde el entorno (`ADMIN_EMAIL`, `ADMIN_PASSWORD`). La
restricción no es una comprobación del endpoint: el puerto de registro **no sabe** crear
administradores, y el que sí sabe no está atado al adaptador HTTP. **La autorización de la ruta es
`admin`, y es lo único que cuelga del router**: ninguna anotación de «anónimo» a nivel de grupo de
rutas; solo `login` es anónimo, y se declara en su propia ruta.

**Petición**

| Campo | Tipo | Obligatorio | Notas |
|---|---|---|---|
| `username` | `string` | **sí** | Se normaliza igual que en login |
| `password` | `string` | **sí** | El hash (argon2) lo calcula el adaptador de seguridad; el dominio nunca ve la clave |
| `role` | `string` | **sí** | Solo `"seller"`. `"admin"` se rechaza con 422 (DP-04); cualquier otro valor, también |

**201 Created**

```json
{"id":"…"}
```

**Sin cabecera `Location`** ([D-C6](#d-c6--register-deja-de-anunciar-location)).

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta un campo o el JSON no casa | §2.2 |
| **401** | Sin token | vacío |
| **403** | Token de `seller` | vacío |
| **422** | `role` es `"admin"` (**DP-04**) | §2.1, `detail` = `"Solo se pueden dar de alta vendedores. El administrador lo crea el despliegue."` |
| **422** | El usuario ya existe | §2.1, `detail` = `"El usuario '<nombre>' ya existe."` |
| **422** | `role` fuera del conjunto | §2.1, `detail` = `"Rol no válido: '<rol>'."` |

> **El orden de comprobación es contrato.** `"admin"` se rechaza **antes** de mirar el nombre: sobre
> un usuario ya ocupado, el mensaje que llega es el de DP-04, no el del duplicado. Para cualquier
> **otro** rol inválido gana el duplicado, porque esa validación vive en el constructor de `User`,
> que se ejecuta después de la búsqueda. Orden: (1) `role == "admin"`, (2) usuario duplicado,
> (3) rol fuera del conjunto.

Dos altas concurrentes del mismo nombre dejan **una sola fila**: lo garantiza el índice único sobre
`user.username`, no la comprobación previa (CA-07.6); la violación de unicidad se traduce al mismo
422 de duplicado.

**Implicación de la intercalación.** Con `utf8mb4_0900_ai_ci` la unicidad **no distingue acentos**:
`jose` y `josé` son el **mismo** usuario, y dar de alta el segundo es el 422 de duplicado. (Las
mayúsculas ya las elimina la normalización.)

---

### E-03 · `GET /api/products`

| | |
|---|---|
| **Autorización** | **Autenticado** (cualquier rol) |

**Parámetros de consulta**

| Parámetro | Tipo | Obligatorio | Por omisión | Notas |
|---|---|---|---|---|
| `search` | `string` | no | — sin filtro | Coincidencia **parcial** («contiene») sobre el nombre, **sin distinguir mayúsculas ni acentos** (CA-01.2: es la intercalación de la base, `utf8mb4_0900_ai_ci`). Los comodines `%` y `_` de la entrada se **escapan**: son texto, no patrón. La técnica de consulta (`LIKE '%texto%'` con escapado) la fija [`data-model.md`](data-model.md) §6.2 |
| `categoryId` | `uuid` | no | — sin filtro | Un valor que no es `uuid` → **400** |
| `page` | entero | no | `1` | [§1.2](#12-paginación) |
| `size` | entero | no | `20` | [§1.2](#12-paginación) y [D-C5](#d-c5--un-size-por-encima-del-máximo-se-recorta-al-máximo) |

**Orden de las filas, como contrato: por `name` ascendente.** Es el patrón de acceso Q1 de
[`data-model.md`](data-model.md) §6.1, y los índices de §6.2 (comprobados en T-13) están dimensionados para servirlo.

**Los productos dados de baja no aparecen nunca** (CA-01.4), ni siquiera filtrando por su categoría.

**200 OK — `PagedResult<ProductView>`**, con `items` de esta forma:

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `id` | `string` (uuid) | no | |
| `name` | `string` | no | Recortado de espacios por el dominio |
| `price` | `number` | no | Dos decimales. **Siempre mayor que cero** |
| `currency` | `string` | no | `"COP"` ([D-C10](#d-c10--currency-nunca-viaja-nulo)) |
| `stock` | `number` | no | Entero, **nunca negativo** |
| `categoryId` | `string` (uuid) | no | |
| `categoryName` | `string` | no | El nombre **vivo** de la categoría. En el catálogo sí se lee vivo; en el reporte **no** (ADR-004) |
| `imageUrl` | `string` \| **`null`** | **sí** | Ruta **relativa** `"/media/<clave>"`. **`null` si el producto no tiene imagen** — nunca una cadena vacía ni una dirección rota (CA-03.2) |

**Atributos que el producto NO tiene, y no es un olvido:** no hay `description`, ni `sku`, ni código
de referencia. **DP-03**: solo los atributos que enumera el enunciado.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | `categoryId`, `page` o `size` no enlazan | §2.2 |
| **401** | Sin token o token inválido | vacío |

---

### E-04 · `GET /api/products/{id}`

| | |
|---|---|
| **Autorización** | **Autenticado** |
| **Ruta** | `{id}`: `uuid`; si no lo es, 404 ([D-C7](#d-c7--un-id-mal-formado-es-404-y-se-documenta-como-tal)) |

**200 OK — `ProductView`**, misma forma que los `items` de [E-03](#e-03--get-apiproducts).

**Devuelve también los productos dados de baja** (ADR-003: la carga por identificador no filtra).
Es lo que permite que el detalle de una venta antigua resuelva su producto.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **401** | Sin token | vacío |
| **404** | El producto no existe | vacío |
| **404** | **`{id}` no es un `uuid`** ([D-C7](#d-c7--un-id-mal-formado-es-404-y-se-documenta-como-tal)) | vacío |

---

### E-05 · `POST /api/products`

| | |
|---|---|
| **Autorización** | **`admin`** |

**Petición**

| Campo | Tipo | Obligatorio | Reglas |
|---|---|---|---|
| `name` | `string` | **sí** | No vacío ni solo espacios. Se recorta |
| `price` | `number` | **sí** | **Mayor que cero**, estrictamente |
| `stock` | `number` | **sí** | Entero **no negativo**. Un decimal → 400 |
| `categoryId` | `uuid` | **sí** | Debe existir (CA-02.4) |

**Tipos estrictos.** El esquema rechaza como 400 lo que pydantic aceptaría en modo laxo: un número
dentro de una cadena (`"price":"12.5"`, `"stock":"3"`), un booleano donde va un número, y un
decimal donde va un entero (`"stock":1.5`; `1.0` también). Es un test, no una opción del
implementador.

**No acepta `currency`** — el sistema es monomoneda (D-05). **No acepta `imageUrl` ni bytes de
imagen**: la imagen va por [E-08](#e-08--post-apiproductsidimage), en una petición aparte. Un
campo desconocido se ignora.

**201 Created**

```json
{"id":"…"}
```

con `Location: /api/products/{id}` — esta sí existe: la ruta de lectura de [E-04](#e-04--get-apiproductsid).

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta un campo, o un tipo no casa (`"price":"abc"`) | §2.2, con `errors.price` |
| **401** | Sin token | vacío |
| **403** | Token de `seller` (CA-02.7) | vacío · [D-C8](#d-c8--el-403-se-mantiene-y-se-exige-la-prueba-que-lo-demuestre) |
| **422** | `name` vacío | §2.1, `"El nombre del producto es obligatorio."` |
| **422** | `price` menor o igual a cero (CA-02.2) | §2.1, `"El precio debe ser mayor a cero."` |
| **422** | `stock` negativo (CA-02.3) | §2.1, `"El stock inicial no puede ser negativo."` |
| **422** | `categoryId` es `00000000-…` | §2.1, `"La categoría es obligatoria."` |
| **422** | `categoryId` no existe (CA-02.4) | §2.1, `"La categoría <id> no existe."` — **mensaje prescrito por este contrato**; T-04 debe escribirlo con ese texto |

**Orden de comprobación**, para que el mensaje sea determinista: primero la forma del JSON (400),
después nombre, precio y categoría en el orden de la fábrica de `Product`, y el stock **al final**
—`Product` valida `stock` después de renombrar, fijar el precio y asignar la categoría—. La
existencia de la categoría se comprueba **antes** de construir el producto.

---

### E-06 · `PUT /api/products/{id}`

| | |
|---|---|
| **Autorización** | **`admin`** |

**Petición.** Idéntica a [E-05](#e-05--post-apiproducts) —`name`, `price`, `stock`, `categoryId`—,
**todos obligatorios**: es un reemplazo completo, no un parche. El `{id}` de la ruta manda; el
cuerpo **no lleva `id`**.

**204 No Content**, sin cuerpo.

**Errores.** Los mismos 422 de E-05, más:

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | El cuerpo no enlaza | §2.2 |
| **401** / **403** | Sin token / `seller` | vacío |
| **404** | No existe, o `{id}` no es `uuid` | vacío |
| **409** | Otra operación cambió la fila y se agotaron los **3 reintentos** de [ADR-002](adr/adr-002-concurrencia-optimista.md) | §2.1, `title` = `"Conflicto con otra operación simultánea"` |

---

### E-07 · `DELETE /api/products/{id}`

| | |
|---|---|
| **Autorización** | **`admin`** |

**204 No Content**, sin cuerpo.

**Es una baja lógica** ([ADR-003](adr/adr-003-baja-logica.md)): la fila no se borra. El producto
desaparece del catálogo y **no se puede vender** (CA-02.6), pero **las ventas que lo contienen
siguen intactas y consultables** (CA-02.5) y **sigue apareciendo en el reporte** del período en que
se vendió (CA-06.3).

**Orden obligatorio cuando el producto tiene imagen** (D-08, CA-03.3): anular la clave de imagen y
**confirmar** antes de borrar el binario. Al revés queda una referencia apuntando a un binario que
ya no existe.

**Repetir la baja.** Volver a dar de baja un producto ya retirado responde **204** otra vez: la
entidad de dominio no conoce la marca de baja —`deleted_at` vive solo en el modelo de persistencia—
y no puede distinguir los dos casos. Efecto colateral declarado: **se reescribe la fecha de baja**.
Esa fecha no forma parte de ningún contrato, así que no rompe nada; si alguna vez importa preservar
la primera, es un ajuste de T-09.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **401** / **403** | Sin token / `seller` | vacío |
| **404** | Ningún producto con ese identificador, o `{id}` no es `uuid` | vacío |
| **409** | Conflicto de concurrencia | §2.1 |

---

### E-08 · `POST /api/products/{id}/image`

| | |
|---|---|
| **Autorización** | **`admin`** |
| **Cuerpo** | `multipart/form-data` |

**Petición**

| Campo | Tipo | Obligatorio | Notas |
|---|---|---|---|
| `file` | fichero | **sí** | **El nombre del campo es exactamente `file`.** Coincide con el `form.append('file', …)` de la app. El parámetro de la ruta se declara `UploadFile` con ese nombre |

**Tipos aceptados**: **`image/jpeg`, `image/png`, `image/webp`**, según el `Content-Type` declarado
de la parte (no se inspecciona el contenido). Cualquier otro se rechaza.

**El nombre del fichero que envía el cliente no se reutiliza nunca**: la clave es un `uuid4().hex`
(32 caracteres sin guiones) más la extensión que corresponde al tipo aceptado. Evita el recorrido de
rutas y las colisiones. El binario se guarda bajo `MEDIA_ROOT`.

**200 OK**

```json
{"url":"/media/9f2c…a1.jpg"}
```

Ruta **relativa**, la misma que aparecerá después en `imageUrl` del producto (CA-03.1).

**Tamaño máximo: 5 MB, con 422** y `detail = "La imagen supera el máximo de 5 MB."`. Lo aplica la
API, **después** de recibir la parte (Starlette no corta el cuerpo por tamaño). Por eso hay un
**techo físico** aparte, que pone el proxy: el `nginx.conf` de la app **debe** fijar
`client_max_body_size 6m` en la `location` de `/api/` —el valor por omisión de nginx es 1 MB y
rechazaría con un 413 HTML una imagen perfectamente válida—. Por encima de 6 MB responde nginx con
su 413 (HTML, **fuera de este contrato**, y sin pasar por la API); entre 5 y 6 MB responde la API
con el 422. Sin proxy no hay techo físico.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta el campo `file` | §2.2, `errors.file` |
| **401** / **403** | Sin token / `seller` | vacío |
| **404** | El producto no existe, o `{id}` no es `uuid` | vacío |
| **422** | Tipo de archivo no permitido | §2.1, `"Tipo de archivo no permitido: <contentType>."` |
| **422** | Supera los 5 MB | §2.1, `"La imagen supera el máximo de 5 MB."` |

> **Aviso para T-04.** Un tipo no permitido o un tamaño excesivo **deben llegar como regla de
> negocio** (422), no como una excepción de infraestructura que acabe en 500: el adaptador de
> almacenamiento (`FileStorage`) no debe lanzar `ValueError` ni similares por esto. El mecanismo
> —validar en el servicio o traducir en el manejador— lo elige T-04; **el código de respuesta no**.

---

### E-09 · `GET /api/categories`

> Es la ficha que cierra la brecha entre la app y la API: el selector de categoría de los
> formularios de producto se pinta con esta respuesta, así que el endpoint **debe existir** desde
> T-04.

| | |
|---|---|
| **Autorización** | **Autenticado** (cualquier rol) |
| **Petición** | Sin cuerpo y **sin parámetros**. No se filtra, no se busca, no se pagina |

**200 OK — array plano**

```json
[{"id":"33333333-3333-4333-8333-333333333333","name":"Electricidad"},
 {"id":"44444444-4444-4444-8444-444444444444","name":"Fontanería"},
 {"id":"11111111-1111-4111-8111-111111111111","name":"General"},
 {"id":"22222222-2222-4222-8222-222222222222","name":"Herramientas"},
 {"id":"55555555-5555-4555-8555-555555555555","name":"Pinturas"}]
```

| Campo | Tipo | Nulo |
|---|---|---|
| `id` | `string` (uuid) | no |
| `name` | `string` | no |

- **Sin envoltorio de paginación**: son datos semilla de la migración `seed_categories`, no un catálogo que
  crezca (D-10). La ruta se declara con `response_model=list[CategoryDto]`.
- **Ordenado por `name` ascendente**, con la intercalación de la base
  ([D-C1](#d-c1--get-apicategories-existe-devuelve-un-array-plano-ordenado-por-nombre)).
- **Vacío es `200 []`**, nunca 204 ni 404.
- **Solo lectura.** No hay `POST`, `PUT` ni `DELETE` de categorías, y no es un olvido:
  [`architecture.md`](architecture.md) §3.1 lo razona.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **401** | Sin token o token inválido | vacío |

---

### E-10 · `POST /api/sales`

| | |
|---|---|
| **Autorización** | **Autenticado** (cualquier rol: vender es de `seller`) |

**Petición**

| Campo | Tipo | Obligatorio | Reglas |
|---|---|---|---|
| `lines` | array | **sí** | **Al menos un elemento**, comprobado por el dominio (422), no por el esquema |
| `lines[].productId` | `uuid` | **sí** | Debe existir y **estar activo** |
| `lines[].quantity` | entero | **sí** | **Mayor que cero**, comprobado por el dominio. Un decimal → 400 |

**El autor de la venta NO va en el cuerpo.** Sale del token: la dependencia que resuelve el usuario
actual lee el claim `unique_name` y se lo pasa al caso de uso. No hay campo `soldBy` en la petición,
y enviarlo no hace nada.

> **Acoplamiento a vigilar.** Si el claim falta o cambia de nombre, la venta **no** debe registrarse
> con un autor de relleno (`"desconocido"`): la dependencia falla con 401. Un test de integración
> hace una venta con el token real de `login` y comprueba que `soldBy` es el nombre de usuario.

**201 Created**

```json
{"id":"…"}
```

con `Location: /api/sales/{id}`.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta `lines`, o el JSON no enlaza | §2.2, `errors.lines` |
| **401** | Sin token | vacío |
| **422** | `lines` vacío (CA-04.3) | §2.1, `"La venta debe tener al menos un ítem."` |
| **422** | Producto repetido en dos líneas (CA-04.4) | §2.1, `"La venta tiene productos repetidos."` |
| **422** | El producto no existe o está dado de baja (CA-02.6) | §2.1, `"El producto <id> no existe."` |
| **422** | `quantity` menor o igual a cero (CA-04.5) | §2.1, `"La cantidad debe ser mayor a cero."` |
| **422** | Stock insuficiente (CA-04.2) | §2.1, `"Stock insuficiente para '<nombre>': disponible <n>, solicitado <m>."` |
| **409** | Dos ventas simultáneas del último ejemplar, agotados los 3 reintentos (CA-04.6) | §2.1 |

**El orden de comprobación es contrato.** Con varias cosas mal a la vez, el mensaje que llega es
el primero de esta lista:

1. `lines` vacío
2. productos repetidos
3. el producto no existe
4. `quantity` menor o igual a cero
5. stock insuficiente

> **Contraintuitivo, y por eso se escribe:** una línea con `quantity: 0` sobre un producto
> inexistente devuelve *«El producto … no existe.»*, **no** *«La cantidad debe ser mayor a cero.»*.
> Quien escriba los tests tiene que saberlo o los escribirá al revés.

**Todo o nada.** Si una sola línea falla, **ningún** producto queda descontado (CA-04.2): la venta y
los descuentos de stock se confirman en la misma unidad de trabajo (`UnitOfWork`).

---

### E-11 · `GET /api/sales`

| | |
|---|---|
| **Autorización** | **Autenticado** |

**Parámetros de consulta:** `from` y `to` **obligatorios**
([§1.3](#13-rangos-de-fecha)), más `page` y `size` ([§1.2](#12-paginación)). En el DTO de consulta,
`from` se declara con alias (`Query(alias="from")`), porque es palabra reservada de Python.

**Orden de las filas, como contrato: por `soldAt` descendente** — la más reciente primero. Es el
patrón Q7 de [`data-model.md`](data-model.md) §6.1.

**200 OK — `PagedResult<SaleView>`**, con `items` de esta forma:

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `id` | `string` (uuid) | no | |
| `soldAt` | `string` | no | ISO 8601 con desplazamiento (`+00:00`) |
| `soldBy` | `string` | no | El **nombre de usuario** normalizado, no un identificador |
| `total` | `number` | no | **Calculado desde las líneas**, nunca almacenado (CA-05.4) |
| `currency` | `string` | no | `"COP"` |
| `items` | `SaleItemView[]` | no | **`[]` si no hay, nunca `null`.** En la práctica nunca está vacío: una venta sin líneas no se persiste |

**`SaleItemView`**

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `productId` | `string` (uuid) | no | |
| `productName` | `string` | no | **Congelado en el momento de la venta** (CA-04.7). Renombrar el producto después **no** lo cambia |
| `quantity` | `number` | no | Entero mayor que cero |
| `unitPrice` | `number` | no | **Congelado**: no sigue al catálogo |
| `subtotal` | `number` | no | `unitPrice × quantity`, calculado |

**El ítem no lleva `currency` propia**: el front usa la de la venta (`Money.of(item.unitPrice,
dto.currency)`). No la añadas: sería una segunda fuente para el mismo dato.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta `from` o `to`, formato no aceptado, o `page`/`size` no enlazan | §2.2 |
| **401** | Sin token | vacío |
| **422** | `to` anterior a `from` (CA-05.3) | §2.1, `"La fecha final no puede ser anterior a la inicial."` |

---

### E-12 · `GET /api/sales/{id}`

| | |
|---|---|
| **Autorización** | **Autenticado** |

**200 OK — `SaleView`**, misma forma que los `items` de [E-11](#e-11--get-apisales) (CA-05.1).

**Una venta registrada no se modifica nunca**: no hay `PUT` ni `DELETE` de ventas, y eso es
deliberado.

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **401** | Sin token | vacío |
| **404** | La venta no existe, o `{id}` no es `uuid` | vacío |

---

### E-13 · `GET /api/reports/sales`

| | |
|---|---|
| **Autorización** | **Autenticado** |

**Parámetros de consulta:** `from` y `to`, **obligatorios**, con las reglas de
[§1.3](#13-rangos-de-fecha). **No se pagina** y **no hay más parámetros**: en particular **no hay
filtro ni agrupación por vendedor** — **DP-02**, para no cruzar datos personales del operador.

**200 OK — `SalesReport`**

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `from` | `string` | no | El extremo recibido, **normalizado a UTC** (`+00:00`). Alias JSON de `from_`; la respuesta se serializa por alias |
| `to` | `string` | no | Ídem |
| `salesCount` | `number` | no | Número de **ventas** del rango, no de líneas ni de unidades |
| `grandTotal` | `number` | no | Total del período. **`0` si no hay ventas**, nunca `null` |
| `currency` | `string` | no | `"COP"` **siempre**, también con el reporte vacío ([D-C10](#d-c10--currency-nunca-viaja-nulo)) |
| `rows` | `SalesReportRow[]` | no | **`[]` si no hay ventas**, nunca `null` (CA-06.2) |

**`SalesReportRow` — una fila por producto y etiqueta de categoría congelada** (CA-06.1)

| Campo | Tipo | Nulo | Qué es |
|---|---|---|---|
| `productId` | `string` (uuid) | no | La clave de agrupación |
| `productName` | `string` | no | **El nombre congelado de la venta más reciente del rango** — **DP-01** |
| `categoryName` | `string` | no | El nombre **congelado** de la categoría en la línea (ADR-004, T-11). **No se consulta el catálogo vivo**, y la agregación agrupa **POR este valor**: un producto recategorizado dentro del rango llega como **dos filas**, una por etiqueta. Nadie elige ganador, porque elegirlo dejaría que una venta nueva cambiara lo que un período cerrado ya dijo |
| `unitsSold` | `number` | no | Suma de cantidades |
| `revenue` | `number` | no | Suma de subtotales congelados |

**Orden de las filas, como contrato: por `revenue` descendente.** Es el patrón Q9 de
[`data-model.md`](data-model.md) §6.1, y es lo que hace útil el reporte: lo que más facturó, arriba.

**Invariantes del reporte que el contrato promete**

- **Estable.** Repetir la consulta de un período cerrado devuelve **exactamente lo mismo**, aunque
  el catálogo haya cambiado (CA-06.4, RNF-02). Se cumple porque nombre, precio y categoría están
  congelados en la línea, no leídos en vivo.
- **Incluye los productos retirados** que se vendieron dentro del rango (CA-06.3).
- **Agrega en el motor.** Traer las ventas a memoria para sumarlas es un defecto, no una
  alternativa (CA-06.5): la agregación vive en `SalesReportQuery`, con `GROUP BY` en SQL.
- **`grandTotal` cuadra con la suma de `revenue` de las filas.**
- **Dos rangos contiguos suman el total del período**, por [D-C2](#d-c2--from-inclusivo-to-exclusivo).

**Errores**

| Código | Cuándo | Cuerpo |
|---|---|---|
| **400** | Falta `from` o `to`, o el formato no se acepta | §2.2 |
| **401** | Sin token | vacío |
| **422** | `to` anterior a `from` | §2.1, `"La fecha final no puede ser anterior a la inicial."` |

---

### E-14 · `GET /health`

| | |
|---|---|
| **Autorización** | **Anónimo**, explícitamente |

**200 OK**

```json
{"status":"ok"}
```

`status` es `string` y su único valor es `"ok"`. **No hay ningún otro código**: si el proceso no
responde, no hay respuesta. No comprueba la base de datos — es prueba de vida del proceso, no de
sus dependencias. Lo usa el healthcheck del compose contra el puerto 8000; nginx no lo proxea.

---

### E-15 · `GET /media/{key}`

| | |
|---|---|
| **Autorización** | **Anónimo, y no se puede proteger** |

**No lo sirve ninguna ruta de FastAPI**: lo sirve `StaticFiles` montado en `/media` sobre el
volumen de binarios (`MEDIA_ROOT`, con `html=False`: sin listado de directorio), bajo la misma URL
que el adaptador `FileStorage` publica dentro de `imageUrl`. Al no ser una ruta enrutada **no pasa
por la dependencia de seguridad**.

- `{key}` es un `uuid4().hex` de 32 caracteres más la extensión. **No se puede adivinar ni
  enumerar**, pero **una URL filtrada sirve la imagen a cualquiera, para siempre** (riesgo aceptado:
  [H-6](#5-huecos-declarados-con-su-dueño)).
- **200** con los bytes y su `Content-Type` (por extensión); **404 vacío** si la clave no existe: el
  manejador de `StarletteHTTPException` de §2.3 tiene que cubrir también al `StaticFiles`.

> **Requisito del `nginx.conf` de la app.** Los tres tipos aceptados (`jpg`, `png`, `webp`) deben
> llegar por `http://localhost:8080/media/…`. Un bloque `location` de expresión regular para
> estáticos (`~* \.(png|jpg|webp)$`) **gana** a un `location /media/` de prefijo simple y serviría
> la petición desde el disco de nginx, devolviendo **su** 404 HTML en vez del de la API. Por eso la
> ruta de `/media/` se declara con **`location ^~ /media/`** (el prefijo preferente corta la
> evaluación de expresiones regulares), con `proxy_pass http://api:8000;` **sin URI**, de modo que
> el prefijo se conserve. La prueba es **la forma del 404**: una clave inexistente responde **404 con
> 0 bytes** —el de la API— y no una página HTML de nginx. Se comprueba con los tres binarios reales
> (200) y con una clave inexistente `.jpg` (404 vacío): sondas P-33 y P-34.

---

## 5. Huecos declarados, con su dueño

Lo que este contrato **decide** está cerrado. Lo que sigue son **decisiones de diseño y riesgos que
el código tiene que cumplir**, cada uno con quien responde de él. Ninguno es una decisión pendiente.

| # | Decisión / riesgo | De dónde sale | Dueño |
|---|---|---|---|
| **H-1** | **El 403 debe demostrarse, no solo declararse.** Hace falta un usuario `seller` en ejecución (creado por el test o por la verificación de extremo a extremo vía `register`) y un test de CA-07.4 sobre las cinco operaciones de administrador. Mientras no exista, el 403 es intención, no comportamiento observado | [D-C8](#d-c8--el-403-se-mantiene-y-se-exige-la-prueba-que-lo-demuestre) | **Propietario.** Lo cubre **T-06** (test de integración) y **T-18** (paso del guion de extremo a extremo) |
| **H-2** | **El 400 debe llevar `detail`** y no filtrar nada interno del marco: ni `type`, ni `input`, ni el nombre de ningún tipo. Es el manejador de `RequestValidationError` del adaptador REST; sin él, FastAPI responde su 422 por omisión | [D-C9](#d-c9--el-400-debe-llevar-detail) | **Propietario.** Se implementa con el primer endpoint (**T-04**) |
| **H-3** | **La app debe convertir el día elegido en `to` exclusivo.** Mandar el extremo final sin ajustar pierde el último día del rango | [D-C2](#d-c2--from-inclusivo-to-exclusivo) | **T-15** (app contra la API real) |
| **H-4** | **Test de contrato sobre el JSON serializado** que afirme `totalPages`, uno por colección paginada. Quitar el campo del DTO no rompe la compilación ni los tests de objeto | [D-C11](#d-c11--totalpages-es-la-parte-más-frágil-del-contrato) | **T-04 y T-07**, las primeras que devuelven un `PagedResult` |
| **H-5** | **El `nginx.conf` de la app debe servir `/media/` con `location ^~ /media/`** y `client_max_body_size 6m` en `/api/`. Sin lo primero, los `.jpg` y `.png` no llegarían por el puerto 8080 y se incumple CA-03.1 | [E-15](#e-15--get-mediakey), [E-08](#e-08--post-apiproductsidimage) | **T-03** (verificación en `verify.sh` del repositorio de infraestructura) y **T-15** (el `nginx.conf` vive en la app) |
| **H-6** | **`GET /media/{key}` es público y no se puede proteger** sin cambiar el mecanismo de servicio | [E-15](#e-15--get-mediakey) | **Propietario.** **Riesgo aceptado**, no de implementación: es lo que hace falta para que la etiqueta `<img>` funcione sin cabeceras |

---

## 6. Qué exige este contrato del código

Mapa de decisión a tarea, para que nadie tenga que deducirlo.

| Decisión | Qué hay que escribir | Tarea |
|---|---|---|
| [D-C1](#d-c1--get-apicategories-existe-devuelve-un-array-plano-ordenado-por-nombre) | Método en el puerto entrante, `CategoryDto`, ruta, y el `ORDER BY name` declarado como contrato en un test | **T-04** |
| [D-C2](#d-c2--from-inclusivo-to-exclusivo) | El predicado `from <= sold_at < to` en las dos consultas, con un test de frontera que meta una venta exactamente en `to` y compruebe que **no** entra | **T-07, T-08** |
| [D-C3](#d-c3--solo-iso-8601-con-desplazamiento-explícito) | Validación estricta del formato (patrón sobre `str`), con test de `2026-01-01` → 400, `01/06/2026` → 400 y epoch → 400 | **T-07, T-08** |
| [D-C4](#d-c4--from-y-to-son-obligatorios-de-verdad) | `from` y `to` sin valor por omisión en las dos rutas y rechazo explícito de la ausencia | **T-07, T-08** |
| [D-C5](#d-c5--un-size-por-encima-del-máximo-se-recorta-al-máximo) | `max_size if size > max_size else (20 if size < 1 else size)` en `PageRequest`, con test | **T-04** |
| [D-C6](#d-c6--register-deja-de-anunciar-location) | `register` devuelve el 201 sin cabecera `Location`, con test | **T-06** |
| [D-C7](#d-c7--un-id-mal-formado-es-404-y-se-documenta-como-tal) | El manejador reencamina a 404 los errores de validación solo de ruta; un test por verbo | **T-04, T-07** |
| [D-C8](#d-c8--el-403-se-mantiene-y-se-exige-la-prueba-que-lo-demuestre) | Usuario `seller` en ejecución y test de CA-07.4 | **H-1 (T-06, T-18)** |
| [D-C9](#d-c9--el-400-debe-llevar-detail) | Manejadores de excepción de §2: 400 con `detail`, vacíos para 401/403/404/405, 500 genérico | **H-2 (T-04)** |
| [D-C10](#d-c10--currency-nunca-viaja-nulo) | Constante de moneda del dominio en el reporte vacío, con test de rango sin ventas | **T-08** |
| [D-C11](#d-c11--totalpages-es-la-parte-más-frágil-del-contrato) | Test de contrato sobre el JSON serializado | **H-4 (T-04, T-07)** |
| **Autenticación por omisión** (§1) | Dependencia de seguridad a nivel de router; `login`, `health` y `media` como únicas excepciones; test que recorre las rutas | **T-06** |
| **Subida de imagen** ([E-08](#e-08--post-apiproductsidimage)) | Tamaño de 5 MB y tipos permitidos como 422; `client_max_body_size` en nginx | **T-04** (API), **T-15** (nginx) |

---

## 7. Firma

**Qué se acepta.** Los **quince endpoints** de §4, con la forma exacta de cada petición y de cada
respuesta, la obligatoriedad y el valor por omisión de cada parámetro, el formato de fecha, la
inclusividad del rango, el orden de las filas de las cuatro colecciones, **todos** los códigos de
error con su cuerpo concreto, y **las once decisiones de §3**, que cierran los ocho huecos
`C-1 … C-8` de [`architecture.md`](architecture.md) §3.1.1 y tres más que el contrato necesita.

**Contra qué se verificará.**

- **El sistema levantado** con el compose (`db`, `api`, `app` *healthy*): las sondas del
  [anexo A](#anexo-a--sondas-de-conformidad-previstas), ejecutadas con `curl` en T-18, con la salida
  literal como constancia.
- **Los tests de contrato** del repositorio `simple-stock-flow-api` sobre el JSON que serializa la
  ruta real, no sobre objetos.
- **Las fuentes ejecutables**: los DTO pydantic de `src/stockflow/adapters/inbound/api/schemas.py`,
  los resultados de `src/stockflow/application/ports/inbound/` y, del lado de la app,
  `src/infrastructure/http/dto/api.dto.ts`, el interceptor de errores, `Money` y el `nginx.conf`.
- **Los criterios de aceptación** de [`spec.md`](spec.md) y las decisiones de negocio **DP-01**,
  **DP-02**, **DP-03** y **DP-04**, que este documento respeta y no reabre.

**Qué queda explícitamente fuera.**

- **El modelo de datos** —tablas, columnas, restricciones, índices, migraciones—: vive en
  [`data-model.md`](data-model.md) (las decisiones técnicas, en [`plan.md`](plan.md)). Aquí solo aparece cuando se ve desde
  fuera, como el orden de las filas o la intercalación.
- **La arquitectura hexagonal, los puertos y los flujos**: [`architecture.md`](architecture.md).
- **Los seis puntos H-1 a H-6** de §5: están **nombrados con su dueño**, no cerrados.
- **Cualquier código.** Este documento **no escribe una sola línea**: especifica.

**Fecha:** 2026-10-02.

---

## Anexo A — Sondas de conformidad previstas

Guion para T-18: cada fila es una petición con `curl` y la respuesta que el contrato exige. La
numeración `P-xx` es estable para poder citarla desde el cuerpo. `$B` es la base (`:8080` salvo
que se indique), `$T` un token de administrador y `$S` uno de `seller`; ni la contraseña ni el
token se pegan en la constancia. Lo que cambia en cada ejecución (token, marcas de tiempo) se
sustituye por `...`.

| # | Petición | Respuesta esperada |
|---|---|---|
| P-01 | `GET :8000/health` | 200 `{"status":"ok"}` |
| P-02 | `GET /api/products` sin token | 401, `Content-Length: 0`, `WWW-Authenticate: Bearer` |
| P-03 | `GET /api/products` con token inválido | 401, vacío, `Bearer error="invalid_token"` |
| P-04 | `GET /api/categories` con `$T` | 200, array plano de cinco, orden `Electricidad … Pinturas` |
| P-05 | `GET /api/products` con `$T` | 200, `items`, `page`, `size`, `total`, **`totalPages`**; `imageUrl` presente aunque sea `null` |
| P-06 | `GET /api/products/no-es-uuid`, y `PUT`, `DELETE` y `POST …/image` con el mismo valor, y `GET /api/sales/no-es-uuid` | 404 vacío en las cinco |
| P-07 | `POST /api/sales` con `{"lines":[]}` | 422 `"La venta debe tener al menos un ítem."` |
| P-08 | `POST /api/sales` sin `lines` | 400, `errors.lines`, `detail` en español |
| P-09 | `POST /api/sales` con JSON roto | 400, forma de §2.2 |
| P-10 | `GET /api/reports/sales` sin `from` ni `to` | 400, `errors.from` y `errors.to` |
| P-11 | `GET /api/reports/sales?from=2026-12-01T00:00:00Z&to=2026-01-01T00:00:00Z` | 422 `"La fecha final no puede ser anterior a la inicial."` |
| P-12 | `…?from=manzana&to=2026-01-01T00:00:00Z` | 400, `errors.from` |
| P-13 | `…?from=2026-01-01&to=2026-12-31` (sin desplazamiento) | 400 |
| P-14 | `…?from=01/06/2026&to=…` | 400 |
| P-15 | `…?from=1767225600&to=1798761600` (epoch) | 400 |
| P-16 | `POST /api/auth/register` sin token | 401 vacío |
| P-17 | `POST /api/auth/register` con `$S` | 403 vacío |
| P-18 | `POST /api/auth/register` con `$T` y `"role":"admin"` | 422 mensaje de DP-04, incluso sobre un usuario ya existente |
| P-19 | `POST /api/auth/register` con `$T`, usuario existente, rol `seller` | 422 `"El usuario '<nombre>' ya existe."` |
| P-20 | `POST /api/auth/register` con `$T`, usuario nuevo | 201 `{"id":…}` y **ninguna** cabecera `Location` |
| P-21 | `POST /api/auth/login` con clave incorrecta | 422 `"Usuario o contraseña incorrectos."` (el mismo mensaje con usuario inexistente) |
| P-22 | `POST /api/auth/login` sin `password` | 400, `errors.password` |
| P-23 | `POST /api/auth/login` con `"password":""` | 422 |
| P-24 | `POST /api/auth/login` con `"  ADMIN  "` y la clave correcta | 200, `username` normalizado, sin hash |
| P-25 | `POST /api/products` con `{"price":"abc",…}` | 400, `errors.price` |
| P-26 | `POST /api/products/{id}/image` sin el campo `file` | 400, `errors.file` |
| P-27 | `POST /api/products/{id}/image` con un PNG de 5,5 MB | 422 `"La imagen supera el máximo de 5 MB."` |
| P-28 | `POST /api/products/{id}/image` con `image/gif` | 422 `"Tipo de archivo no permitido: image/gif."` |
| P-29 | `GET /api/products?size=999` | 200, `"size": 100` |
| P-30 | `GET /api/products?size=0` y `?size=abc` | `0` → 200 con `"size": 20`; `abc` → 400 |
| P-31 | `GET :8000/media/<clave inexistente>.jpg` sin token | 404, `Content-Length: 0` |
| P-32 | Venta con una línea `quantity: 0` sobre un producto inexistente | 422 `"El producto … no existe."` (no el de cantidad) |
| P-33 | `GET $B/media/<clave real>` para `.jpg`, `.png` y `.webp` | 200 con el `Content-Type` de la imagen, los tres |
| P-34 | `GET $B/media/<clave inexistente>.jpg` | 404 **con 0 bytes** (el de la API, no el HTML de nginx) |
| P-35 | Producto repetido en dos líneas de la misma venta | 422 `"La venta tiene productos repetidos."` |
| P-36 | `GET /api/auth/login` | 405 vacío con `Allow: POST` |
| P-37 | `GET /api/ruta-inexistente` y `GET /api/ruta-inexistente.jpg` con `$T` | 404 vacío; **nunca** `{"detail":"Not Found"}` ni el HTML de nginx |
| P-38 | Con `$S`: `POST`, `PUT`, `DELETE` de producto, `POST …/image` y `register` | 403 vacío en las cinco; las cuatro lecturas dan 200 y `POST /api/sales` llega a la regla de negocio |
| P-39 | `GET /api/reports/sales` de un rango sin ventas | 200 `{"salesCount":0,"grandTotal":0,"currency":"COP","rows":[]}` |
| P-40 | Una venta exactamente en `to` | **no** entra en el rango; sí entra una exactamente en `from` |
| P-41 | Venta con el token real de `login` | `soldBy` es el nombre de usuario, no un valor de relleno |
| P-42 | Cualquier ruta salvo `login`, `health` y `media` sin token | 401 |
