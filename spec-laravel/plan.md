# Plan técnico — Manager Library (Laravel)

> **Fuente normativa:** [`../spec-python/plan.md`](../spec-python/plan.md). Este documento traduce
> **solo** las decisiones de tecnología. El negocio, el contrato HTTP y el modelo de datos no se
> tocan.

Cómo se construye lo que dicen [`../spec-python/spec.md`](../spec-python/spec.md) y
[`constitution.md`](constitution.md). El esquema **no** se describe aquí: vive en
[`../spec-python/data-model.md`](../spec-python/data-model.md). La forma del sistema, en
[`architecture.md`](architecture.md).

## 1. Decisiones técnicas

| # | Decisión | Elección (Laravel) | Por qué |
|---|---|---|---|
| D-01 | Lenguaje y runtime | PHP 8.3+ con Laravel 12 | Stack del enunciado; Laravel 12 es la versión estable de la línea 11/12 |
| D-02 | Framework HTTP | Laravel 12: `routes/api.php` con prefijo `/api` + `Route::prefix('v1')` | Contrato en [`../spec-python/api-contract.md`](../spec-python/api-contract.md), base `/api/v1` |
| D-03 | Persistencia | Eloquent + **migraciones escritas a mano** sobre PostgreSQL 16 | El esquema es de las migraciones (ADR-001). El scaffolding de Laravel viola `data-model.md` §3: ver [INC-02](consistencia.md) |
| D-04 | Concurrencia | **Compare-and-swap** sobre `available_copies` + `CHECK` de rango | RNF-2 sin añadir columna. [ADR-005](adr/adr-005-compare-and-swap.md) |
| D-05 | Fechas | Instantes UTC (`timestamptz`); el reloj es un puerto `Clock` | Probar vencimientos sin esperar 14 días. D-05 y [ADR-007](adr/adr-007-reloj-como-puerto.md) |
| D-06 | Reporte | Una consulta agregada por un puerto `ReportReader` | Estable y barato ([ADR-004](../spec-python/adr/adr-004-reporte-agregado-y-congelado.md)) |
| D-07 | Objetos de valor | `final readonly class` dentro de `app/Domain`; sin tabla | No tienen identidad |
| D-08 | Portadas | Puerto `CoverStorage`; adaptador sobre `Storage::disk('local')`; la base guarda solo `cover_key` | Clave opaca, nunca ruta ni bytes |
| D-09 | Contraseñas | **Argon2id** tras el puerto `PasswordHasher`; `config/hashing.php` con `driver => 'argon2id'` | Artículo IX. El default de Laravel es **bcrypt**: hay que cambiarlo |
| D-10 | Arranque | La aplicación crea el admin inicial desde `ADMIN_USERNAME` / `ADMIN_PASSWORD` | La base no puede producir el hash |
| D-11 | Autenticación | **JWT HS256** con `firebase/php-jwt`, 8 h, `JWT_SECRET` | Simple y **sin estado**, sin tablas nuevas. [ADR-006](adr/adr-006-jwt-sin-estado.md) |
| D-12 | Pruebas | **Pest** (compatible con PHPUnit), fakes de los puertos, PostgreSQL real solo en integración | Pirámide de pruebas |
| D-13 | Dependencias de arquitectura | **deptrac** + tests propios en `tests/Architecture` | Sustituye a `import-linter`, que no existe para PHP ([INC-13](consistencia.md)) |
| D-14 | Errores HTTP | Un único manejador global con el sobre del contrato | RNF-4. [ADR-008](adr/adr-008-cuerpo-de-error-unico.md) |

## 2. Configuración por entorno

| Variable | Obligatoria | Ejemplo | Nota |
|---|---|---|---|
| `DB_CONNECTION` | sí | `pgsql` | Laravel no usa `DATABASE_URL` |
| `DB_HOST` / `DB_PORT` | sí | `db` / `5432` | Servicio de `infra` |
| `DB_DATABASE` | sí | `library` | |
| `DB_USERNAME` / `DB_PASSWORD` | sí | `library` / `library` | Solo en el entorno, nunca en el repo |
| `JWT_SECRET` | sí | cadena larga aleatoria | Artículo IX |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | sí | definidos en `.env`, nunca en el repo | D-10 |
| `COVERS_DIR` | no | `./var/covers` | D-08 |
| `CORS_ORIGIN` | sí | `http://localhost:5173` | Origen único del frontend |
| `CLOCK_OFFSET_DAYS` | no | `20` | **Solo** para el sembrador de `tool`. [ADR-007](adr/adr-007-reloj-como-puerto.md) |

`config/database.php` fija además `'timezone' => 'UTC'` y `'search_path' => 'library,public'` en la
conexión `pgsql`: RN-6 exige instantes UTC, y el esquema es `library` (`data-model.md` §0).

## 3. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué | Dónde |
|---|---|---|---|
| Dominio | Invariantes: máximo 3, vencimiento a 14 días, disponibles en rango | Nada: objetos puros | `tests/Unit/Domain` |
| Aplicación | Casos de uso: prestar, devolver, reporte | **Fakes** de los puertos | `tests/Unit/Application` |
| Integración | Repositorios y consulta del reporte | PostgreSQL real | `tests/Integration` |
| Contrato | Cuerpo de error único y códigos de estado | La API levantada | `tests/Contract` |
| Concurrencia | Dos préstamos del último ejemplar | PostgreSQL real | `tests/Integration/Concurrency` |
| Esquema | Las consultas de `data-model.md` §10 | PostgreSQL real | `tests/Integration/Schema` |
| Arquitectura | Artículos I, II y III | `deptrac` + tests propios | `tests/Architecture` |

## 4. Orden de construcción

Esqueleto y salud → migración inicial con semilla → dominio → casos de uso con fakes → adaptadores
→ API → composición y arranque → invariantes al motor → pruebas de arquitectura. El detalle, con
evidencia, está en [`tasks.md`](tasks.md).

Un único detalle de este orden: `page` (T-23) no depende de nada y puede construirse en paralelo
desde el principio; `infra` (T-20) escribe su esqueleto pronto pero cierra cuando la API levante.
