# Simple Stock Flow — Arquitectura Onion · VERSIÓN FINAL

---
▓▓▓ BLOQUE 1 / 29 ▓▓▓

> **Documento de arranque del equipo.** Define el qué, el cómo y el orden de trabajo.
> Quien lea esto completo no necesita volver a preguntar qué construir.
>
> **Antes de empezar:** hay que tener abierta la especificación en
> `test-simple-stock-flow-docs/spec-python/`. Este documento **traduce** esa
> especificación a Laravel + React en Onion. No la reemplaza: es la segunda
> fuente de verdad si se contradicen, y por eso toda decisión de traducción
> está registrada como ADR.

---

▓▓▓ BLOQUE 2 / 29 ▓▓▓

## 📌 1 · Qué construimos

**Simple Stock Flow** — control de stock y ventas para un almacén pequeño.

El sistema tiene **3 afirmaciones** y todo se apoya en ellas:

1. El stock que muestra el catálogo **es** el stock que hay.
2. Una venta registrada **no se puede alterar** después.
3. El reporte de un período cerrado dice hoy lo mismo que dirá en un año.

**Actores:** `seller` (vende y consulta) · `admin` (todo lo anterior + catálogo, imágenes y alta de usuarios) · anónimo (solo login y health).

**Historias:** HU-01 catálogo · HU-02 mantener catálogo · HU-03 imagen · HU-04 registrar venta · HU-05 consultar ventas · HU-06 reporte · HU-07 autenticación · HU-08 operar el sistema.

**No existe el comprador como actor.** Está fuera de alcance: devoluciones, pagos, envíos, descuentos, impuestos, notificaciones y CRUD de categorías.


---
▓▓▓ BLOQUE 3 / 29 ▓▓▓

## 📌 2 · Las 12 reglas de negocio

Cada una tiene dueño, capa que la prueba y tarea asignada.

| # | Regla | Tarea |
|---|---|---|
| RN-01 | El stock **nunca** es negativo | T-10 · T-20 |
| RN-02 | El precio es **mayor que cero** | T-05 · T-21 |
| RN-03 | La cantidad es **mayor que cero** | T-21 |
| RN-04 | Una venta tiene **al menos una línea** | T-10 |
| RN-05 | Un producto **no se repite** en la misma venta | T-10 · T-20 |
| RN-06 | El precio y el nombre se **congelan** al vender | T-11 |
| RN-07 | Una venta registrada **no se modifica ni se anula** | T-22 |
| RN-08 | Un producto vendido **no se borra**: se da de baja | T-09 |
| RN-09 | Todos los importes en la **misma moneda** | T-05 |
| RN-10 | `username` **único**, normalizado a minúsculas | T-06 · T-21 |
| RN-11 | El rol está en el conjunto cerrado `{admin, seller}` | T-21 |
| RN-12 | El total **siempre** es la suma de sus líneas | T-07 |

**RN-07 se cumple por ausencia:** no hay operación de edición en el agregado, ni método en el puerto, ni verbo HTTP. T-22 lo demuestra revisando las tres superficies.


---
▓▓▓ BLOQUE 4 / 29 ▓▓▓

## 📌 3 · Las 4 decisiones cerradas de negocio

| # | Pregunta | Decisión | Consecuencia técnica |
|---|---|---|---|
| **DP-01** | Producto renombrado entre dos ventas del mismo rango: ¿qué nombre muestra el reporte? | El **congelado más reciente dentro del rango** | Ventana en la consulta agregada del motor. El catálogo vivo no se consulta nunca |
| **DP-02** | ¿El reporte desglosa por vendedor? | **No.** Solo por producto | `SalesReportRow` es cerrado. Ningún endpoint acepta filtro por vendedor |
| **DP-03** | ¿El catálogo necesita descripción, código o SKU? | **No.** Exactamente 5 atributos: nombre, precio, stock, categoría, imagen | `product` no tiene columnas más allá de esos 5 + `deleted_at` + `version` |
| **DP-04** | ¿Un admin puede crear otros admins? | **No.** Los admins crean sellers | **Estructural:** el puerto de registro no sabe crear admins |


---
▓▓▓ BLOQUE 5 / 29 ▓▓▓

## 📌 4 · Contrato de API — los 15 endpoints

`api-contract.md` **manda**. Si el OpenAPI autogenerado discrepa, el código está mal.

| # | Método | Ruta | Auth | Caso de uso |
|---|---|---|---|---|
| E-01 | POST | `/api/auth/login` | anónimo | `Authenticate` |
| E-02 | POST | `/api/auth/register` | **admin** | `Authenticate` |
| E-03 | GET | `/api/products` | autenticado | `ManageProducts` |
| E-04 | GET | `/api/products/{id}` | autenticado | `ManageProducts` |
| E-05 | POST | `/api/products` | **admin** | `ManageProducts` |
| E-06 | PUT | `/api/products/{id}` | **admin** | `ManageProducts` |
| E-07 | DELETE | `/api/products/{id}` | **admin** | `ManageProducts` |
| E-08 | POST | `/api/products/{id}/image` | **admin** | `ManageProducts` |
| E-09 | GET | `/api/categories` | autenticado | `ManageProducts` |
| E-10 | POST | `/api/sales` | autenticado | `PlaceSale` |
| E-11 | GET | `/api/sales` | autenticado | `GetSales` |
| E-12 | GET | `/api/sales/{id}` | autenticado | `GetSales` |
| E-13 | GET | `/api/reports/sales?from=&to=` | autenticado | `GetSalesReport` |
| E-14 | GET | `/health` | anónimo | — |
| E-15 | GET | `/media/{key}` | anónimo | `ManageProducts` |

**5 puertos entrantes, no 6:** el alta de usuario (E-02) cae dentro de `Authenticate`. El recuento lo fija el spec y no cambia.


---
▓▓▓ BLOQUE 6 / 29 ▓▓▓

## 📌 5 · Los 3 cuerpos de error (no negociable)

Solo pueden salir estas tres formas.

| Forma | Cuándo | Cuerpo |
|---|---|---|
| **422 / 409 / 500** | regla de negocio, conflicto, excepción | `application/problem+json` con `detail` en **español** |
| **400** | forma mal formada | `{title, status, detail, errors:{campo:[msg]}}`. El 500 filtra nada: ni tipo, ni traza, ni SQL |
| **401 / 403 / 404 / 405** | auth, permiso, inexistente, método | **Cuerpo vacío.** `Content-Length: 0`. Se conservan `WWW-Authenticate` y `Allow` |

**Regla que lo gobierna:** el esquema valida **solo la forma** (presencia, tipo, formato). **Toda invariante de negocio la valida el dominio** y sale como **422**. En los DTO está **prohibido** `gt`, `ge`, `min_length` sobre campos de negocio: moverían un 422 a un 400.

⚠️ El **422 por defecto de Laravel hay que reemplazarlo entero.**


---
▓▓▓ BLOQUE 7 / 29 ▓▓▓

## 📌 6 · ARQUITECTURA ONION — las 5 capas

**4 anillos + 1 punto de ensamblaje.**

```
        ┌──────────────────────────────────────┐
        │  Bootstrap                           │  ← imports todo.
        │  app/Bootstrap/                      │    NADIE lo importa.
        │  PortBindingsServiceProvider.php     │    NO ES UN ANILLO.
        └───────────────┬──────────────────────┘
                        │ conecta implementaciones con sus interfaces
     ┌──────────────────┴──────────────────┐
     ▼                                     ▼
┌─────────────────┐            ┌──────────────────────┐
│ Presentation    │            │ Infrastructure       │
│ Anillo 4        │            │ Anillo 3             │
│ controllers     │            │ Eloquent · mappers   │
│ requests        │            │ repos · JWT · binarios│
│ resources       │            │ migraciones · config │
│ errores         │            └──────────┬───────────┘
│ ═══ PROHÍBIDO ═══│                       │
│ importar Infra  │                       │
└────────┬────────┘                       │
         │        imports hacia adentro   │
         ▼                                ▼
       ┌───────────────────────────────────────┐
       │ Application   Anillo 2                │
       │ casos de uso · 5 puertos in           │
       │ 10 puertos out · SIN lógica negocio   │
       └──────────────────┬────────────────────┘
                          │ solo importa Domain
                          ▼
       ┌───────────────────────────────────────┐
       │ Domain   Anillo 1                    │
       │ entidades · value objects · invariantes│
       │ IMPORTA NADA. Ni Laravel. Ni la BD.   │
       └───────────────────────────────────────┘
```


---
▓▓▓ BLOQUE 8 / 29 ▓▓▓

## 📌 7 · Las 6 reglas de dependencia y cómo se verifican

Cada regla tiene un mecanismo. Una regla sin comprobación es una intención.

| # | Regla | Verificación |
|---|---|---|
| **R-01** | `Domain` no importa nada del proyecto ni de terceros | `grep -R "Illuminate\\\\" app/Domain` → **0**. Un test carga las clases **sin bootear Laravel** |
| **R-02** | `Application` solo importa `Domain` | Deptrac |
| **R-03** | `Presentation` **no** importa `Infrastructure` | Deptrac |
| **R-04** | Todo constructor de `Application\` recibe solo clases de `Domain` o puertos de `Outbound` | test de reflexión |
| **R-05** | Todo constructor de `Presentation\` recibe solo puertos de `Inbound` — **nunca** de `Outbound` | test de reflexión |
| **R-06** | Solo `app/Bootstrap/` instancia `Infrastructure\*` | test de reflexión |

**R-03, R-04 y R-05 son las que hacen real la Onion.** Sin ellas, un controlador puede pedir un repositorio inyectado y saltarse el caso de uso: el negocio pasa a estar en el router.

**Herramientas:** **Deptrac** (capas) · **PHPStan** nivel max + Larastan (tipos) · **PHPUnit 11** (pruebas). Equivalen a `import-linter` + `mypy --strict` + `pytest` del spec.


---
▓▓▓ BLOQUE 9 / 29 ▓▓▓

## 📌 8 · Por qué los puertos viven en Application y no en Domain

**Hay desacuerdo entre compañeros y está resuelto. Queda por escrito.**

La Onion clásica de Palermo pone las interfaces de repositorio dentro de `Domain`. Nuestra constitución no:

> **Artículo II** — «Todo lo que necesita del exterior lo expresa como un puerto que **él mismo define** en `ports/outbound/`.»
> **Artículo IV (DIP)** — «Los módulos de `application/ports/outbound/` viven en el paquete de aplicación.»

Además es **verificable**: un test comprueba que el puerto está en `Application` y no en `Domain`. Si lo movemos, el instructor lo ve en diez segundos.

**Se quedan en `Application/Ports/`.** Registrado como **ADR-005**.


---
▓▓▓ BLOQUE 10 / 29 ▓▓▓

## 📌 9 · Backend — Domain (Anillo 1)

```
app/Domain/
├── Model/
│   ├── Product.php        raíz de agregado · SIN `version`
│   ├── Sale.php           raíz de agregado · SIN `total`
│   ├── SaleItem.php       entidad interna · SIN `subtotal`
│   ├── Category.php       entidad de referencia, solo lectura
│   └── User.php           raíz de agregado
├── ValueObject/
│   ├── Money.php          Brick\Math\BigDecimal · escala 2 · HALF_UP
│   ├── Quantity.php       entero > 0
│   ├── ProductId.php · SaleId.php · CategoryId.php · UserId.php
│   ├── Username.php       normaliza a minúsculas (RN-10)
│   └── Role.php           admin | seller (RN-11)
├── Exception/             mensajes en ESPAÑOL · viajan tal cual al 422
│   ├── BusinessRuleViolation.php        ← base abstracta
│   ├── InsufficientStockException.php · InvalidPriceException.php
│   ├── InvalidQuantityException.php · EmptySaleException.php
│   ├── RepeatedProductException.php · ProductNotFoundException.php
│   ├── UnknownCategoryException.php · DuplicateUsernameException.php
│   └── InvalidRoleException.php · InvalidCredentialsException.php
└── Service/               VACÍA hasta que una regla se gane el lugar
    └── .gitkeep
```

⚠️ **`Domain/Service/` empieza vacío a propósito.** El Artículo VI fuerza toda invariante al agregado y T-05 pone la guarda de moneda **dentro** de `Product`. Un anillo vacío para rellenar un círculo de un dibujo es lo que el Artículo X prohíbe.

⚠️ **Aquí no puede aparecer `Illuminate`.** Ni `use`, ni facades, ni `now()`, ni `config()`, ni `extends Model`.


---
▓▓▓ BLOQUE 11 / 29 ▓▓▓

## 📌 10 · Backend — Application (Anillo 2)

```
app/Application/
├── Ports/
│   ├── Inbound/           ← 5 puertos + DTOs (fuente del contrato)
│   │   ├── PlaceSale.php · ManageProducts.php · GetSales.php
│   │   ├── GetSalesReport.php · Authenticate.php
│   │   ├── PlaceSaleCommand.php
│   │   ├── ProductView.php · SaleView.php · SaleItemView.php
│   │   └── SalesReport.php · SalesReportRow.php
│   │   └── AuthResult.php · PagedResult.php
│   └── Outbound/          ← 10 puertos
│       ├── ProductRepository.php · SaleRepository.php
│       ├── CategoryRepository.php · UserRepository.php
│       ├── FileStorage.php · PasswordHasher.php · TokenGenerator.php
│       └── Clock.php · UnitOfWork.php · SalesReportQuery.php
├── UseCase/               un caso de uso = una clase
│   ├── PlaceSaleService.php       (T-10)
│   ├── ProductCatalogService.php  (T-04)
│   ├── GetSalesService.php        (T-07)
│   ├── SalesReportService.php     (T-08)
│   └── AuthenticationService.php  (T-06)
├── Model/
│   ├── PageRequest.php    el recorte a 100 es regla de aplicación
│   └── DateRange.php       `from` inclusivo · `to` exclusivo
└── Exception/
    └── ConcurrencyConflict.php   ← la define Application (ADR-002)
```

**Los nombres coinciden con `tasks.md`.** Cada archivo mapea 1:1 contra una tarea. El instructor no necesita traducción.

**En `Application` NO hay:** `DB::`, `Eloquent`, `Illuminate`, ni un `if` de negocio. `Application` orquesta; el dominio decide.


---
▓▓▓ BLOQUE 12 / 29 ▓▓▓

## 📌 11 · El puerto `UnitOfWork` — no inventar otro

El spec **ya tiene** este puerto (Artículo II, entre los 10 salientes). Un compañero propuso llamarlo `TransactionManagerInterface`; es el mismo puerto con otro nombre.

**Se llama `UnitOfWork`**, y adopta la forma con `callable`:

```php
interface UnitOfWork {
    public function run(callable $operation): mixed;
}
```

**Por qué `callable` y no `begin/commit/rollback`:** con un `callable` es **imposible** olvidar el commit o dejar una transacción abierta.

La implementación `LaravelUnitOfWork` es la única que contiene `DB::transaction()`. Eso es todo lo que R-04 comprueba.


---
▓▓▓ BLOQUE 13 / 29 ▓▓▓

## 📌 12 · Backend — Infrastructure (Anillo 3)

```
app/Infrastructure/
├── Persistence/
│   ├── Model/             Eloquent · NUNCA en Domain
│   │   └── ProductModel.php · SaleModel.php · SaleItemModel.php
│   │       CategoryModel.php · UserModel.php
│   ├── Mapper/            ProductMapper.php · SaleMapper.php · …
│   │                      ← aquí vive `version`, que el dominio NO ve
│   ├── Repository/
│   │   ├── EloquentProductRepository.php · EloquentSaleRepository.php
│   │   ├── EloquentCategoryRepository.php · EloquentUserRepository.php
│   │   └── EloquentSalesReportQuery.php   ← agrupa EN EL MOTOR (ADR-004)
│   ├── LaravelUnitOfWork.php           ← DB::transaction() vive AQUÍ
│   └── Concurrency/       0 filas afectadas → ConcurrencyConflict
├── Security/
│   └── JwtTokenGenerator.php · Argon2PasswordHasher.php
├── Storage/
│   └── LocalFileStorage.php   clave opaca · el dominio nunca ve rutas
├── Configuration/
│   └── Settings.php     readonly · SIN default para secretos (Art. IX)
└── Logging/
    └── CorrelationId.php
```

**El Mapper sí vale la pena.** No está para «hacer bonito»: está para que **Eloquent pueda cambiar sin obligar al dominio a cambiar**.

**`version` es el caso que lo prueba.** La tabla tiene `version`; `Product` **no lo tiene**. El mapper genera `UPDATE … WHERE id = ? AND version = ?`; si afecta 0 filas, traduce a `ConcurrencyConflict`. El dominio no sabe que existe la concurrencia.


---
▓▓▓ BLOQUE 14 / 29 ▓▓▓

## 📌 13 · Backend — Presentation (Anillo 4)

```
app/Presentation/
├── Http/
│   ├── Controller/        8 controladores
│   ├── Request/           SOLO forma. Prohibido gt/min en campos de negocio
│   ├── Resource/          ProductResource · SaleResource · …
│   ├── Serialization/     CamelCase · Money(→number) · Utc(→+00:00)
│   └── ProblemDetails/    los 3 cuerpos de error del contrato
│       ├── ProblemDetailsRenderer.php    422 / 409 / 500
│       ├── ValidationErrorRenderer.php   400 con `errors`
│       └── EmptyErrorRenderer.php        401/403/404/405 · Content-Length: 0
└── Middleware/
    └── AuthenticateToken.php · RequireRole.php · CorrelationIdMiddleware.php

app/Bootstrap/                             ← NO ES UN ANILLO
└── PortBindingsServiceProvider.php         ← único archivo que amarra un puerto

routes/api.php    solo URL → controlador. Cero lógica.
database/migrations/   initial_schema + seed_categories
```

**Las rutas se quedan en `routes/api.php`** (convención de Laravel) y solo enlazan. `Bootstrap` se llama así porque Laravel ya tiene un `bootstrap/` raíz: el nombre no pelea con el framework. `bootstrap/providers.php` tiene **una sola línea**.


---
▓▓▓ BLOQUE 15 / 29 ▓▓▓

## 📌 14 · Frontend — los mismos 4 anillos

```
app/src/
├── domain/              Anillo 1 · sin React
│   ├── model/           Product · Cart · SaleItem · Money · Quantity
│   └── error/
├── application/         Anillo 2
│   ├── ports/           ProductRepository · CartRepository · SessionRepository
│   ├── use-cases/       BrowseCatalog · AddToCart · Checkout · Login · ViewSalesReport
│   │                    ← testeable SIN navegador, SIN red, SIN servidor
│   └── state/
├── infrastructure/      Anillo 3
│   ├── http/
│   │   ├── client.ts · interceptors/
│   │   └── dto/api.dto.ts   ← fuente ejecutable del contrato (la nombra el spec)
│   ├── mappers/
│   └── providers.ts        ← composition root del front
└── features/            Anillo 4 = PRESENTACIÓN
    └── auth/ catalog/ cart/ sales/ reports/
```

**En React no hace falta inventar `Presentation`: las pantallas YA son el anillo exterior.** Por eso se llama `features/`.

**Regla:** una pantalla **no** importa `infrastructure/http/`. Llama a un caso de uso. Si una pantalla llama al cliente HTTP directo, la política del carrito se salta — que es justo lo que el §2.1 de `architecture.md` quiere poder probar sin servidor.

⚠️ **Una estructura tipo `components/ pages/ services/ hooks/` NO sustituye estas cuatro.** Son la plantilla de Vite. Puede convivir como carpeta `shared/`, pero las tres del spec no se omiten.


---
▓▓▓ BLOQUE 16 / 29 ▓▓▓

## 📌 15 · Repos que NO llevan Onion

| Repo | Por qué |
|---|---|
| **`infra`** | Solo contenedores, red, volúmenes y MySQL vacío. **Cero lógica de negocio.** Ojo: el repo `infra` **no es** la capa `Infrastructure` |
| **`tool`** | Es un **cliente HTTP** de la API. No conoce puertos ni dominio. **Nunca** se conecta a MySQL |
| **`page`** | HTML/CSS estático. El spec le **prohíbe** hablar con la API |
| **`docs`** | Documentación |

**Resultado: 2 repos con Onion** (`api` y `app`), **4 sin nada de Onion**. Escribirlo aquí evita volver a discutirlo.


---
▓▓▓ BLOQUE 17 / 29 ▓▓▓

## 📌 16 · Todo en Docker — sin excepciones

**Nada de PHP, Node, MySQL ni Composer en la máquina.** Todo dentro de contenedores.

```
infra/docker-compose.yml            ← el orquestador
  ├── db      MySQL 8.4 · motor VACÍO · sin una línea de DDL
  ├── api     Laravel · aplica migraciones al arrancar · :8000
  ├── app     nginx sirve el build de React + proxy de /api y /media · :8080
  └── [perfil dev] composer · node    ← instalan dependencias
```

**Las 6 carpetas van como HERMANAS**, no anidadas. El compose construye con `../`:

```yaml
build:
  context: ../test-simple-stock-flow-api      # ojo: prefijo test-
  context: ../test-simple-stock-flow-app
```

**4 trampas de Docker que ya están resueltas:**
1. `vendor/` y `node_modules/` como **volúmenes con nombre**, no carpetas montadas (en Windows rompe permisos).
2. Los repos de compose se llaman `test-simple-stock-flow-*`.
3. `.env.example` con **todas** las claves y **cero valores**. El README dice `cp .env.example .env` como paso 1, o el evaluador no levanta nada.
4. `verify.sh` corre **dentro** de un contenedor: `docker compose run --rm api bash verify.sh`.


---
▓▓▓ BLOQUE 18 / 29 ▓▓▓

## 📌 17 · Los 10 errores que más caro salen

| # | Error | Consecuencia |
|---|---|---|
| 1 | `Illuminate`, facades o `now()` en `Domain` | Bandera roja inmediata. Es el pecado capital |
| 2 | `class Product extends Model` en `Domain` | Eloquent se vuelve el centro |
| 3 | `DB::transaction()` dentro del caso de uso | Application conoce Laravel |
| 4 | Un controlador pide un repositorio inyectado | El negocio se salta al router (R-05) |
| 5 | `version` aparece en la entidad `Product` | Rompe ADR-002 |
| 6 | Columnas `total` o `subtotal` en la base | Rompe Artículo VII: dos fuentes de verdad |
| 7 | `migrations/` dentro de `infra` | Rompe Artículo V y ADR-001 |
| 8 | El **422** por defecto de Laravel | El contrato exige 3 formas de error distintas |
| 9 | Seeder normal en vez de migración `seed_categories` | Sin UUID fijos, las pruebas no pueden referenciar |
| 10 | El admin inicial sembrado desde SQL | D-10 lo prohíbe: solo el puerto de hash puede producir su hash |

**Verificación rápida de los más graves:**
```bash
grep -R "Illuminate\\\\"  app/Domain        # debe dar 0
grep -R "DB::"            app/Application   # debe dar 0
grep -R "Infrastructure"  app/Presentation  # debe dar 0
find infra -name "*.php"                   # debe dar 0
```


---
▓▓▓ BLOQUE 19 / 29 ▓▓▓

## 📌 18 · Riesgos técnicos que hay que cerrar antes de codificar

Estos **no** están resueltos por el spec porque son propios de PHP.

| Riesgo | Decisión tomada |
|---|---|
| **PHP no tiene tipo decimal.** El spec exige `Decimal` y prohíbe `float` | **`Brick\Math\BigDecimal`** en `Money`. Escala 2, `HALF_UP`. El mapper **rechaza** >2 decimales en vez de redondear en silencio |
| **Ni Alembic ni el schema builder de Laravel generan `CHECK`** | Los **9 `ck_*`** se escriben a mano con `DB::statement`. Si desaparecen, RN-01/02/03/10/11 quedan solo en el dominio |
| **`totalPages` es obligatorio y derivado** | Si se modela como `@property` sin declararlo como campo, **no serializa**, no peta PHPStan, y el paginador deja de dibujar páginas. Requiere test de contrato contra el **HTTP serializado**, no contra el objeto |
| **`currency` nunca puede ser `null`** | El reporte vacío lo lleva desde `Money`. Si llega `null`, la app hace `toUpperCase()` y **deja en blanco** la página |
| **`imageUrl` debe sobrevivir como `null`** | Prohibido filtrar nulos. Nunca cadena vacía, nunca URL rota |
| **Fechas: `DATETIME(6)` en UTC, sin zona** | El mapper **rechaza** un `datetime` ingenuo. Un instante sin zona desplaza los rangos del reporte en silencio |
| **Concurrencia sin reintento** | El reintento es del **caso de uso completo desde la lectura**, máx. **3**, y después **409**. Reintentar solo la escritura reaplicaría un descuento calculado sobre stock viejo |


---
▓▓▓ BLOQUE 20 / 29 ▓▓▓

## 📌 19 · Los ADRs que hay que escribir

| ADR | Decisión |
|---|---|
| **005** | Por qué Onion y no hexagonal — y por qué 4 anillos + Bootstrap |
| **006** | Por qué los puertos viven en `Application` y no en `Domain` |
| **007** | `Brick\Math\BigDecimal`: PHP no tiene decimal |
| **008** | Deptrac como equivalente de `import-linter` |
| **009** | Las trampas de Laravel: facades, Active Record y auto-wiring del contenedor |
| **010** | Docker: `vendor` y `node_modules` como volúmenes; compose reemplaza Testcontainers |

⚠️ **Cada vez que nos desviamos del spec a propósito hay que declararlo.** Divergimos en: Onion, React con capas, PHPUnit en vez de pytest, compose en vez de Testcontainers, `BigDecimal` en vez de `Decimal`.

> *«Una mentira declarada es deuda. Una mentira silenciosa es una trampa.»* — Artículo X.3


---
▓▓▓ BLOQUE 21 / 29 ▓▓▓

## 📌 20 · Workflow — cómo trabajar los 6 chats sin cruzarse

**Principio:** cada chat tiene **una carpeta de repo** y trabaja ahí. Físicamente no puede escribir fuera.

```
                    ┌────────────────────────┐
                    │  docs/      Chat A      │ ← ÚNICO que escribe aquí
                    │  arquitectura Onion     │
                    │  plan-laravel · ADRs    │
                    │  CONTRATO DTO CONGELADO│
                    └────────────┬───────────┘
                          solo LECTURA
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
      ┌─────────┐        ┌─────────┐        ┌─────────┐
      │  api    │        │  app    │        │  tool   │
      │ Chat B  │        │ Chat C  │        │ Chat D  │
      └─────────┘        └─────────┘        └─────────┘
      + page: Chat D (suelto, cero acoplamiento)
                             │
                             ▼   ← último de todos
                       ┌─────────┐
                       │  infra  │
                       │ Chat E  │
                       └─────────┘
```

**4 reglas:**
1. **Nada se decide dos veces.** Los chats B, C y D **transcriben** de `docs/`. No deciden arquitectura, ni nombres de campo, ni textos de error.
2. **Un dueño por ruta.** Ningún chat toca una ruta de otro.
3. **Puertas de entrada.** Un chat no arranca hasta que su prerrequisito existe.
4. **Si falta un dato, se para y se agrega a `docs/` primero.**

**Paralelismo real: 2×, no 6×.** `api` bloquea a `app` y a `infra`. `infra` espera a todos. 6 chats existen, pero 4 arrancan después del primero.


---
▓▓▓ BLOQUE 22 / 29 ▓▓▓

## 📌 21 · El brief que se le pega a cada chat

```
CONTEXTO
Proyecto: Simple Stock Flow (prueba técnica SENA ADSO 3413974).
Lee PRIMERO ../test-simple-stock-flow-docs/spec-python/, en este orden:
constitution.md · spec.md · plan.md · architecture.md
· data-model.md · api-contract.md · tasks.md · adr/
ARQUITECTURA: Onion de 4 anillos (Domain, Application, Infrastructure,
Presentation) + Bootstrap. Decidida y documentada. No la discutas.

TU REPO
SOLO test-simple-stock-flow-<REPO>
Tu directorio de trabajo YA es esa carpeta. No salgas de ella.
PUEDE ESCRIBIR: <rutas, una por línea>. Cualquier otra: no la toques.

NO DEBES DECIDIR (ya está en docs/): arquitectura · nombres de campo del
contrato → transcribe de api-contract.md · textos de error en español →
transcribe · estructura de la base → transcribe de data-model.md

REGLAS INNEGOCIABLES (constitution.md)
· Todo en Docker. Nunca asumas PHP, Node ni MySQL en la máquina.
· Art. VI: si hay un `if` de negocio, está en el lugar equivocado.
· Art. VII: lo derivado se calcula, nunca se guarda.
· Art. VIII: primero el test que falla, después el código.
· Art. XI: código, comentarios, nombres de test y tablas en INGLÉS.
          READMEs y docs/ en ESPAÑOL.
· Art. IX: cero secretos. Si falta una variable, el arranque falla ruidosamente.

AL TERMINAR: reporta qué rutas creaste y qué commands usaste para comprobarlo.
```


---
▓▓▓ BLOQUE 23 / 29 ▓▓▓

## 📌 22 · Ramas — regla de commits

```
main                        ← siempre revisable
 └─ docs/onion-decision     ← P0 · SOLO documentación
 └─ phase/1-domain
 └─ phase/2-schema-infra
 └─ phase/3-use-cases
 └─ phase/4-robustness
 └─ phase/5-app
 └─ phase/6-satellites
 └─ phase/7-delivery
```

**Regla:** dentro de cada rama, **el commit de documentación es el primero**. Después, código + tests juntos (Art. VIII).

**El PR de `docs` se mergea antes que el de código.** Eso es la regla de «docs primero», hecha demostrable en el historial.

**Flujo de un PR:**
1. Chat escribe en su rama
2. `git push -u origin <rama>`
3. Se abre el PR en GitHub
4. Se mergea a `main` cuando el test pasa


---
▓▓▓ BLOQUE 24 / 29 ▓▓▓

## 📌 23 · Fases de ejecución

| Fase | Qué | Repos | Puerta |
|---|---|---|---|
| **P0** | Arquitectura Onion + `plan-laravel.md` + ADR-005…010 + **contrato DTO congelado campo por campo** | `docs` | — |
| **P1** | Imagen Docker · Deptrac + PHPStan + PHPUnit · `tests/Architecture/` **escrito primero y fallando** · `Domain/` | `api` | P0 mergeado |
| **P2** | `initial_schema` + `seed_categories` con los **9 CHECK a mano** · `verify.sh` **antes** del compose | `api` `infra` | P1 |
| **P3** | Casos de uso test-first: catálogo (T-04) · moneda (T-05) · auth (T-06) · consultar ventas (T-07) · reporte (T-08) | `api` | P2 |
| **P4** | Barreras del motor (T-20) · baja lógica (T-09) · `PlaceSale` + concurrencia (T-10) · categoría congelada (T-11) · autoría (T-12) · índices (T-13) · invariantes (T-21) · venta intocable (T-22) | `api` | P3 |
| **P5** | React: login, catálogo CRUD, carrito, ventas, reporte contra la API real | `app` | contrato congelado |
| **P6** | `page` estática · seeder vía API · logging estructurado | `page` `tool` `api` | P4 |
| **P7** | Los 3 READMEs · publicación · `verify.sh` de punta a punta · barrido de idioma | todos | P5 · P6 |

**El orden de sacrificio**, si hay que recortar: T-12 → T-14 → T-13 → T-23. **No** se recortan T-09, T-10 ni T-11: son las tres que sostienen las tres afirmaciones del problema.


---
▓▓▓ BLOQUE 25 / 29 ▓▓▓

## 📌 24 · Tres niveles de prueba (no se cruzan)

| Nivel | Qué prueba | Qué **no** puede usar |
|---|---|---|
| `tests/Domain/` | invariantes puras | **ningún** doble de prueba |
| `tests/Application/` | casos de uso | fakes de puertos · **cero** `Infrastructure` · Laravel **no** se bootea |
| `tests/Infrastructure/` | mapeo, filtros, unicidad, concurrencia, reporte | — |
| `tests/Architecture/` | R-01 a R-06 | — |

**La prueba que prueba el diseño entero:** `tests/Application/` corre con fakes y **sin bootear Laravel**. Si un caso de uso no se puede probar así, es que se le coló infraestructura. Esa es la prueba más barata de que la Onion es real.


---
▓▓▓ BLOQUE 26 / 29 ▓▓▓

## 📌 25 · Aviso: `GUIA-ONION.md` tiene un error factual

Dice: *«Ambas describen una arquitectura **Onion**.»*

**Es falso.** El spec es **hexagonal**. `architecture.md` se titula **«El hexágono del servicio»**, define `adapters/inbound/api` y `adapters/outbound/{persistence,storage,security}`, y `arquitectura-panoramica.md` Figura 6 dice literalmente **«ESTA ES EL DIAGRAMA QUE HAY QUE INVERTIR»**.

**Corrección:** *«Ambas describen una arquitectura **hexagonal**; esta edición traduce esas decisiones a **Onion**.»*

Si lo dejamos escrito, el instructor abre `architecture.md` y ve que no cuadramos con nuestra propia documentación. El Artículo X.3 lo dice: *«Una mentira declarada es deuda. Una mentira silenciosa es una trampa.»*

**Lo que sí está bien en el guide:** el nombre **`Bootstrap`** para el punto de ensamblaje, y las 5 capas que describe.


---
▓▓▓ BLOQUE 27 / 29 ▓▓▓

## 📌 26 · Cómo usar este documento en Discord

Discord admite **~2.000 caracteres** por mensaje. Los **29 bloques** de arriba caben en **29 mensajes separados**.

**Cómo pegarlos:**
1. Abre este archivo en el editor
2. Copia **desde** la línea `▓▓▓ BLOQUE 1` **hasta** justo antes de `▓▓▓ BLOQUE 2`
3. Pégalo en Discord
4. Repite con el bloque 2, y así sucesivamente

Cada bloque es independiente: se entiende sin haber leído los anteriores, salvo el 1.

**Para el DM:** los bloques **6, 7, 17 y 18** son los que nadie debería saltarse. Son la arquitectura y los errores que cuestan más caro.


---
▓▓▓ BLOQUE 28 / 29 ▓▓▓

## 📌 27 · Pendiente de confirmar

| # | Pregunta | Quién decide |
|---|---|---|
| 1 | **El instructor revisa sobre `main` o sobre ramas por tarea?** Afecta la regla de commits | instructor |
| 2 | **El hexagonal que hay que convertir es el del spec, o esperaban construir primero una versión hexagonal?** No hay código hexagonal en ningún repo | instructor |
| 3 | Falta confirmar el **nombre del campo** con el que se crea el admin inicial (`ADMIN_EMAIL` en el spec) | equipo |

**Mientras no se responda la 2, se trabaja en la lectura A:** traducir el spec y construir en Onion desde el inicio.


---
▓▓▓ BLOQUE 29 / 29 ▓▓▓

## 📌 28 · Siguiente paso inmediato

| Paso | Qué | Quién |
|---|---|---|
| **0** | Clonar los 6 forks como carpetas **hermanas** con su `.git` (hoy están doblemente anidados y sin git) | Chat A |
| **1** | `git switch -c docs/onion-decision` en `docs` | Chat A |
| **2** | **Commit de documentación**: este documento + `plan-laravel.md` + ADR-005…010 + corregir `GUIA-ONION.md` + **contrato DTO congelado campo por campo** | Chat A |
| **3** | PR a `main` y merge | Chat A |
| **4** | **Solo entonces** empieza el código | Chats B–E |

**Regla de los pasos 1–3:** no se escribe ni una línea de código antes de que el PR de `docs` esté mergeado.


**Fin del documento.** 28 secciones · 29 bloques de Discord.
Spec completa: `test-simple-stock-flow-docs/spec-python/`