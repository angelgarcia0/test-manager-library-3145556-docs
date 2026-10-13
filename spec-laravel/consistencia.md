# Consistencia del spec y su traducción a Laravel

Análisis de coherencia entre [`spec.md`](../spec-python/spec.md),
[`data-model.md`](../spec-python/data-model.md),
[`api-contract.md`](../spec-python/api-contract.md), [`plan.md`](../spec-python/plan.md),
[`architecture.md`](../spec-python/architecture.md), [`tasks.md`](../spec-python/tasks.md) y los
cuatro ADR, al traducirlos a PHP con Laravel.

**Tensión de partida.** `spec.md`, `data-model.md`, `api-contract.md` y `adr/001..004` son
**byte-idénticos** entre `spec-python/` y `spec-.net/`. Solo difieren `constitution.md`,
`plan.md`, `architecture.md`, `arquitectura-panoramica.md` y `tasks.md`. Es decir: el negocio es
único y lo único que cambia por pila son las decisiones de construcción. Por eso las incoherencias
que siguen son de **traducción al stack**, no del producto.

Severidad: 🔴 bloqueante · 🟠 alta · 🟡 media · ⚪ baja.

| ID | Sev. | Título | Conflicto |
|---|---|---|---|
| INC-01 | 🔴 | El testigo de versión de ADR-002 no existe en el modelo de datos | `adr-002` y `plan.md` D-04 frente a `data-model.md` §3 |
| INC-02 | 🔴 | Los defaults del scaffolding de Laravel violan el modelo de datos | Laravel frente a `data-model.md` §0, §3, §8 |
| INC-03 | 🔴 | Un token con estado crea una séptima tabla | D-11 y artículo VII frente a `data-model.md` §2 |
| INC-04 | 🔴 | T-21 exige tres vencidos generados solo por la API, y CA-06.6 lo impide | `tasks.md` T-21 frente a CA-06.6 y `data-model.md` §9 |
| INC-05 | 🟠 | CA-03.3 exige un 422 sin código en la tabla del contrato | CA-03.3 frente a `api-contract.md` §2 |
| INC-06 | 🟠 | `throttle:api` devuelve 429, un estado inexistente en el contrato | Laravel frente a `api-contract.md` §1 |
| INC-07 | 🟠 | Una portada de 3 MB daría 413, no el 422 de T-15 | `php.ini` frente a `tasks.md` T-15 y CA-03.4 |
| INC-08 | 🟠 | RNF-4 exige un cuerpo único pero no hay código para 5xx | RNF-4 frente a `api-contract.md` §2 |
| INC-09 | 🟡 | `from`/`to` son fechas, no instantes | `api-contract.md` §1 frente a §3.6 y §3.7 |
| INC-10 | 🟡 | «Un 403 no revela si el recurso existe» exige un orden de middleware | `api-contract.md` §4 frente a §1 |
| INC-11 | 🟡 | El índice `(category_id, title)` no cubre el patrón Q1 real | `data-model.md` §4 frente a RNF-1 y Q1 |
| INC-12 | 🟡 | CA-06.4 y CA-06.5 compiten con el socio con tres vencidos | `spec.md` HU-06 |
| INC-13 | 🟡 | `Clock` debe vivir en la aplicación, pero el dominio lo consume | Artículos I y IV frente a `tasks.md` T-05 |
| INC-14 | ⚪ | `PUT /books/{id}` no incluye `isbn`; `size > 50` y `page < 1` sin regla | `api-contract.md` §3.4 y §1 |
| INC-15 | ⚪ | `DELETE` sin código explícito; `days_late` sin redondeo | `api-contract.md` §1 y §3.4 |

---

## 1. Las cuatro bloqueantes

### INC-01 · El testigo de versión no existe en el modelo de datos

**Contexto.** ADR-002 (aceptada), `plan.md` D-04 y `architecture.md` §4 exigen que «el libro lleva
un testigo de versión; al guardar, si la versión cambió, la operación falla y el servidor responde
**409 `BOOK_NOT_AVAILABLE`**», y el diagrama de secuencia muestra `L-->>S: ok o conflicto de
versión`. Pero `data-model.md` §3 fija **exactamente nueve columnas** en `book` y ninguna es
`version`, `row_version` ni `lock_version`. Además §3 prohíbe cualquier `DEFAULT` y §10 exige que
`information_schema.columns` devuelva «las columnas de §3».

La alternativa que ADR-002 descarta —`SELECT … FOR UPDATE`— es justo la que Laravel ofrece de serie
con `lockForUpdate()`: seguir la ruta cómoda rompería una decisión ya tomada.

**Mitigación — `available_copies` es el testigo (compare-and-swap).** Concurrencia optimista sin
columna nueva:

```sql
UPDATE library.book
   SET available_copies = :nuevo
 WHERE id = :id
   AND available_copies = :esperado   -- testigo implícito
   AND available_copies > 0;
-- 1 fila  -> éxito (RNF-2 cumplido: solo uno ganó)
-- 0 filas -> conflicto de versión -> 409 BOOK_NOT_AVAILABLE
```

En PostgreSQL con `READ COMMITTED` la condición del `WHERE` se reevalúa **después** de obtener el
bloqueo de fila: la semántica es CAS real, y dos transacciones que leen `1` y escriben `0` producen
exactamente una fila actualizada. `ck_book_available_range` sigue siendo, como exige ADR-002, la
última barrera ante escrituras fuera del adaptador. Decisión completa en
[`adr/adr-005-compare-and-swap.md`](adr/adr-005-compare-and-swap.md).

**Consecuencias de implementación.**

| Regla | Detalle |
|---|---|
| El valor esperado vive en el **adaptador** | `BookRepository` guarda el `available_copies` leído en el `SELECT`; el `Book` de dominio queda mudo de persistencia (artículo I) |
| Prohibido `increment()` / `decrement()` | `SET x = x - 1` no detecta el conflicto y prestaría un ejemplar inexistente: es la forma más sutil de romper RNF-2 |
| Reintento único opcional | ADR-002 lo autoriza: releer y reintentar una vez solo si sigue habiendo ejemplar |
| Devoluciones: CAS sobre `returned_at` | `UPDATE library.loan_item SET returned_at = now() WHERE id = :id AND returned_at IS NULL`; cero filas = `ITEM_ALREADY_RETURNED` (CA-07.2) |

**Verificación.** T-13 / RNF-2: dos `POST /loans` simultáneos sobre un libro con un ejemplar →
exactamente un 201 y un 409, con `available_copies = 0` al final.

---

### INC-02 · Los defaults de Laravel violan el modelo de datos

`php artisan make:model` y `make:migration` generan exactamente lo que `data-model.md` prohíbe.
Cada fila es una violación literal de una afirmación verificable del documento.

| Default de Laravel | Lo que el spec exige | Acción obligatoria |
|---|---|---|
| `$table->id()` → `bigint … AS IDENTITY` | `id uuid` (§3, §0) | `$table->uuid('id')->primary()` + `HasUuids` |
| `$table->timestamps()` | «no hay columnas de auditoría» (§3 y §8) | `$table->timestamps(false)` y `public $timestamps = false;` en los seis modelos |
| `timestamps()` / `softDeletes()` → `timestamp` | «todas las marcas de tiempo son `timestamptz`» (§3) | `$table->timestamptz(...)` en las cuatro columnas de tiempo |
| `$table->string('isbn')` → `varchar(13)` | `isbn char(13)` | `$table->char('isbn', 13)` |
| `->default(now())` habitual | «Ninguna columna lleva `DEFAULT`» (§3) | Ningún `->default()` en las seis tablas |
| Esquema `public` | `Esquema library` (§0, ADR-001) | `CREATE SCHEMA IF NOT EXISTS library` en la migración + `search_path` en la conexión |
| `$table->unique('isbn')` | isbn único **parcial** `WHERE deleted_at IS NULL` (ADR-003) | Índice parcial con `DB::statement`; un único normal rompería el ISBN re-registrable |
| `CHECK` inexistente en `Blueprint` | `ck_{tabla}_{regla}` (§0) | `DB::statement('ALTER TABLE … ADD CONSTRAINT ck_… CHECK (…)')` |
| Sesión Postgres en zona local | RN-6, instantes UTC | `'timezone' => 'UTC'` en `config/database.php` |

**Mitigación.** Migraciones **escritas a mano**, nunca generadas por scaffolding, y un test de
arquitectura que falle si aparece cualquier `->id()`, `timestamps()` sin `false`, `->default(`
o `bigint` en `app/Database/Migrations/`.

---

### INC-03 · Un token con estado crea una séptima tabla

D-11 pide «JWT HS256, 8 horas, clave en `JWT_SECRET`. **Simple y sin estado**», y
`api-contract.md` §4 exige «JWT firmado, 8 horas, con `sub` y `role`. Sin datos personales
dentro». Pero las dos opciones más conocidas en Laravel crean esquema:

| Paquete | Tabla que crea | Veredicto |
|---|---|---|
| `laravel/sanctum` | `personal_access_tokens` | ❌ séptima tabla, token opaco almacenado, contradice «sin estado» |
| `tymon/jwt-auth` | `token_in_blacklistable_tokens` | ❌ séptima tabla aunque la blacklist esté apagada: la migración existe |
| `firebase/php-jwt` | ninguna | ✅ puro, HS256, sin estado |

`data-model.md` §2 dice «Seis entidades, seis tablas. No hay tabla de reporte, de multas ni de
auditoría», y §8 prohíbe además columnas de auditoría (Sanctum registra `last_used_at`).

**Mitigación.** `firebase/php-jwt`. Detalle en
[`adr/adr-006-jwt-sin-estado.md`](adr/adr-006-jwt-sin-estado.md).

---

### INC-04 · T-21 es imposible de cumplir usando solo la API

`data-model.md` §9 prohíbe expresamente que el sembrador use SQL: «el sembrador de datos de
demostración (`tool`) usa la **API**, nunca SQL directo». T-21 exige que, tras ejecutarlo,
`/loans/overdue` devuelva **3 líneas**. Pero CA-06.6 fija `due_at = loaned_at + 14 días` con
`loaned_at` = instante actual, y `POST /loans` no admite ningún parámetro de fecha: un cliente que
solo habla HTTP no puede producir ni un préstamo vencido sin esperar 14 días.

**Mitigación — el puerto `Clock`, que existe para esto.** `plan.md` D-05 lo anticipa: «el reloj es
un puerto (`Clock`) — probar vencimientos sin esperar 14 días», y el artículo II bendice el puerto
sin que el núcleo sepa quién lo implementa. Detalle en
[`adr/adr-007-reloj-como-puerto.md`](adr/adr-007-reloj-como-puerto.md).

---

## 2. Las altas y las medias

### INC-05 · CA-03.3 exige un 422 sin código en el contrato

CA-03.3 pide 422 cuando ajustar el total dejaría disponibles por debajo de cero, pero
`api-contract.md` §2 no lista ningún código para ese caso y su 409 está reservado a «duplicado, sin
ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos». Un implementador
razonable concluiría que debe inventar un código 409, contradiciendo el AC.

La regla equivalente y no ambigua: **`total_nuevo ≥ total_actual − available_actual`** (no se puede
bajar el total por debajo de los ejemplares prestados).

**Mitigación.** Reutilizar `VALIDATION_ERROR` (422) con `details`; **no inventar código**. El
artículo XI fija el vocabulario en inglés y el contrato es la fuente.

```json
{
  "code": "VALIDATION_ERROR",
  "message": "No se puede reducir el total de ejemplares: hay 3 libros prestados.",
  "details": [{ "field": "total_copies", "issue": "el total no puede ser menor que los ejemplares prestados" }]
}
```

Misma política para `total_copies < 1` y para un `role` distinto de `librarian` (CA-02.3).

### INC-06 · `throttle:api` introduce un 429 inexistente

El grupo `api` de Laravel trae `throttle:api` por defecto; al superarse el límite emite **429**, un
estado que no existe en `api-contract.md` §1. RNF-4 exige el cuerpo único para todo error HTTP.

**Mitigación.** Declarar el grupo `api` **sin** `throttle`. No añadir rate limiting: no lo pide
ningún requisito y `architecture.md` §6 es explícito —«se añaden si un requisito lo pide, no antes».

### INC-07 · Una portada de 3 MB daría 413, no el 422 de T-15

T-15 fija la evidencia: «Subir PNG de 2 MB ok; **3 MB → 422**». Pero los límites de PHP se aplican
**antes** de que la aplicación vea la petición: al superarlos, PHP/FPM responde 413 o trunca el
cuerpo, y la aplicación nunca puede devolver 422.

**Mitigación.** Subir los límites de PHP **por encima** del umbral de negocio para que la validación
de la aplicación sea la que decide: `upload_max_filesize=8M`, `post_max_size=8M`,
`memory_limit=256M`. La regla de 2 MB vive en el puerto `CoverStorage` y devuelve 422. El tipo se
valida **por contenido** (`finfo`, `getimagesize`), nunca por extensión. Para
`DELETE /books/{id}/cover` se respeta el orden obligatorio de `data-model.md` §7: anular `cover_key`
y **confirmar la transacción**, y solo después borrar el binario.

### INC-08 · RNF-4 exige un cuerpo único pero no hay código para 5xx

RNF-4 dice «todo error HTTP usa el mismo cuerpo». La tabla §2 cubre solo códigos de negocio. ¿Qué
`code` poner ante un `QueryException` que dispara el `CHECK` —que ADR-002 declara *defecto*, no caso
de negocio—, un 404 de ruta inexistente o un 500?

**Mitigación.** Un único manejador global con la tabla de mapeo de
[`adr/adr-008-cuerpo-de-error-unico.md`](adr/adr-008-cuerpo-de-error-unico.md) y **sin filtrar
nunca** el mensaje del driver: el contrato prohíbe trazas, SQL y rutas del servidor.

### INC-09 · `from`/`to` son fechas, no instantes

§1 establece «fechas ISO 8601 en UTC, con `Z`», pero §3.7 documenta `"from": "2026-10-01"` y §3.6
usa los mismos parámetros. Tampoco se define la inclusividad de `to`.

**Mitigación (interpretación, no cambio de contrato).** `from` y `to` son **fechas de calendario
UTC** en formato `YYYY-MM-DD`, ambos extremos **inclusivos**, aplicados a `loan.loaned_at`, con
intervalo semiabierto para no perder el último segundo:

```sql
loaned_at >= :from 00:00:00Z  AND  loaned_at < (:to + 1 día) 00:00:00Z
```

`from > to` → 422 `INVALID_DATE_RANGE`; `from = to` es un rango válido de un día.

### INC-10 · 403 vs 404: el orden de middleware decide

§4: «un 403 no revela si el recurso existe». Si un `librarian` llama `DELETE /books/{id}` con un id
inexistente, la respuesta debe ser **403**. Pero el route model binding de Laravel resuelve el
modelo antes de entrar al controlador y aborta con 404.

**Mitigación.** Orden fijo e inviolable: **autenticación → autorización por rol → resolución del
recurso**. Un middleware `EnsureRole` en la ruta antes de la sustitución de parámetros. Los
controladores no revalidan el rol. Test: `DELETE /books/{uuid-inexistente}` con token de
`librarian` → 403.

### INC-11 · El índice de acceso no cubre el patrón Q1 real

`data-model.md` §4 declara `book (category_id, title)` parcial sobre activos. Pero Q1 busca texto
parcial en título **o autor**, y la ordenación por título solo usa ese índice cuando hay filtro de
categoría (`category_id` es la columna líder).

**Por qué no es bloqueante.** RNF-1 exige menos de 300 ms con 5 000 libros; un `ILIKE '%texto%'`
sobre 5 000 filas es de milisegundos. **No se debe reaccionar con precipitate.**

**Mitigación, en este orden y sin tocar el modelo de datos.**

1. Implementar **exactamente** los índices documentados: uno adicional también violaría
   `data-model.md`, y §10 verifica el catálogo completo.
2. Medir con el sembrador (`tool`, `--books=5000`) y **publicar la evidencia** de latencia p50/p95,
   como exige el artículo XII.
3. Solo si la evidencia mostrara incumplimiento, la corrección es **documental y explícita**: una
   ADR y una actualización de `data-model.md` §4 y §10, decididas por el propietario del documento
   (artículo X). No se añade el índice por iniciativa propia.
4. Escapar `%` y `_` en `q` antes del `ILIKE`.

### INC-12 · Dos criterios compiten en el mismo caso

Un socio con **3 libros vencidos** que pide uno más viola a la vez CA-06.4 (máx. 3 → 422) y CA-06.5
(tiene vencidos → 409). El contrato exige 422 y 409 para la misma petición.

**Mitigación.** La Constitución remite al documento de arquitectura, y `architecture.md` §4 ya lo
resuelve: el orden del diagrama es `¿tiene vencidos? ¿supera 3?` → **primero vencidos (409
`MEMBER_HAS_OVERDUE`), después el límite (422 `MEMBER_LIMIT_EXCEEDED`)**. RN-3 es la restricción más
fuerte y 409 es un conflicto de *estado*, no de validación de entrada. Se implementa en ese orden y
se cubre con un test que fija ambos casos.

### INC-13 · `Clock` debe vivir en la aplicación, pero el dominio lo consume

El artículo IV afirma «los puertos viven en el **paquete de aplicación**» y el artículo II lista el
reloj entre las cosas que la aplicación expresa como puerto. Pero T-05 exige «pruebas con un
`Clock` **falso**»: es el **dominio** el que calcula el estado, y si importara `Clock` violaría el
artículo I. Además, en PHP `Carbon` es una librería de terceros y el artículo I la prohíbe en el
dominio igual que a SQLAlchemy.

**Mitigación — el dominio recibe el instante, no el reloj.** El puerto vive en la aplicación
(artículos II y IV intactos) y **quien lo consume** lo invoca y pasa el valor:

```php
$now  = $this->clock->now();                     // DateTimeImmutable
$loan = Loan::start($memberId, $userId, $now);  // el dominio no conoce el reloj
$loan->status($this->clock->now());             // open | overdue | closed
```

El dominio trabaja solo con `DateTimeImmutable`. La evidencia de T-05 se cumple igual: las pruebas
de dominio inyectan un instante fijo y las de aplicación usan un `FakeClock`.

---

## 3. Las bajas

| Punto | Ambigüedad | Decisión adoptada (sin cambiar el contrato) |
|---|---|---|
| `PUT /books/{id}` no lista `isbn` | ¿El ISBN es inmutable? | Inmutable vía API: solo `POST` lo fija. El índice único parcial sigue válido y el ISBN dado de baja puede re-registrarse (ADR-003) |
| `size > 50`, `page < 1` | §1 dice «`size` máximo 50»; §1 dice 422 para «regla de entrada rota» | **Recortar** a 50 y 1, y devolver el valor efectivo en la respuesta; documentar en el README de `api` |
| `DELETE /books/{id}` y `DELETE /books/{id}/cover` | §1 ofrece 204 pero no asigna código | **204 No Content**. `DELETE /cover` sin portada → 204 idempotente; sobre libro dado de baja → 404 |
| `GET /members` sin orden | §3.5 no lo dice | `ORDER BY full_name ASC`, por simetría con el orden por título de HU-04 |
| `days_late` sin redondeo | «días de retraso» no está definido | `floor((now − due_at) / 86400)` en UTC; la fila ya incluye `due_at` para desambiguar |
| Orden por título con acentos | No se especifica collation | La collation por defecto del motor (ICU en `postgres:16`); se fija la configuración regional del contenedor, sin crear objetos de esquema |
| Reporte sin paginación | Devuelve `{from,to,rows}`, no el sobre `{items,page,size,total}` | **Respetar el sobre del reporte**, sin paginar: el contrato no lo pide y añadir paginación lo alteraría |
| `active_loans` en `GET /members/{id}` | «activo» evoca baja lógica; aquí es «sin devolver» | Se mantiene el nombre del contrato y se documenta que significa «líneas sin devolver» |

---

## 4. Sospechas que el spec ya resuelve

Verificados contra las dos versiones: **no** son inconsistencias.

| Sospecha | Por qué no es problema |
|---|---|
| `due_at` está almacenado y el artículo VII prohíbe lo derivado | El artículo VII prohíbe solo **`status` y `days_late`**. `due_at` es un hecho de negocio ya ocurrido, como `loaned_at` y `returned_at` (`data-model.md` §8) |
| Las dos invariantes marcadas `pendiente (T-18)` violan el artículo VI | No: el propio artículo VI lo autoriza —«lo que no está aplicado se marca **pendiente**; no se promete». El mecanismo de honestidad documental es el artículo VI, no una excepción |
| `user` es palabra reservada en SQL | El documento ya lo resuelve: «se escribe `library."user"`; el ORM lo entrecomilla solo» |
| El reporte puede devolver dos filas del mismo `book_id` | Es intencional (ADR-004, punto 4): si un libro cambió de categoría en el rango, el reporte muestra **dos filas** y no elige ganador. La UI de React usa la clave `book_id + category_name` |
| CA-06.4 «más de 3» frente a RN-2 «máximo 3» | Coinciden: el estado válido tiene 3; el cuarto dispara 422 |

---

## 5. Preguntas abiertas

Ninguna bloquea la implementación; todas se resuelven con el criterio documentado arriba.

| # | Pregunta | Criterio adoptado | Quién decide si cambia |
|---|---|---|---|
| P-1 | ¿Se actualiza la marca de `ck_book_total_positive` y `ck_user_role` en `data-model.md` §4 tras T-18? | **No se toca el spec.** La evidencia vive en el repo `api` | El propietario del documento, en un cambio aislado |
| P-2 | `size > 50`: ¿recorte o 422? | Recorte, con el valor efectivo en la respuesta | El propietario del contrato |
| P-3 | ¿`GET /reports/most-borrowed` admite paginación? | No; se respeta la forma del contrato y se documenta el techo de filas | El propietario del contrato |
| P-4 | Si RNF-1 no se cumpliera con los índices documentados | ADR nueva + actualización de `data-model.md` §4 y §10 | El propietario del documento (artículo X) |
