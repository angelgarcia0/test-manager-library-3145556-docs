# Matriz de trazabilidad — del requisito al commit

> Documento propio. **No modifica** el spec: lo recorre para dejar constancia de que cada
> requisito de [`../spec-python/spec.md`](../spec-python/spec.md) tiene una tarea, una evidencia
> ejecutable y una rama donde se sube.

## 1. Historias y criterios

| US | Criterio | Tarea | Evidencia ejecutable | Rama | Tipo de commit |
|---|---|---|---|---|---|
| HU-01 | CA-01.1 vigencia de 8 h | T-14 | Token expirado tras 8 h → 401; `expires_in: 28800` | `feature/T-14-seguridad-argon2id-y-jwt` | `feat` |
| HU-01 | CA-01.2 mensaje genérico | T-07, T-16 | Usuario inexistente y clave mala devuelven el mismo `message` | `feature/T-16-http-contrato-y-errores` | `test` |
| HU-01 | CA-01.3 401 sin token, 403 por rol | T-16 | Recorrido de las 20 rutas sin token y con rol insuficiente | `feature/T-16-http-contrato-y-errores` | `test` |
| HU-02 | CA-02.1 crea `librarian` | T-07 | `CreateLibrarianServiceTest` | `feature/T-07-casos-de-uso-auth` | `feat` |
| HU-02 | CA-02.2 normalizado y único | T-07, T-12 | `"Ana "` → `ana`; repetido → 409 `USERNAME_TAKEN` | `feature/T-12-persistencia-libros-socios-usuarios` | `feat` |
| HU-02 | CA-02.3 `admin` → 422, sin fuga | T-07, T-14 | `role: admin` → 422; ni respuesta ni log contienen la clave | `feature/T-14-seguridad-argon2id-y-jwt` | `test` |
| HU-03 | CA-03.1 disponibles = total | T-04, T-08 | `CreateBookServiceTest` | `feature/T-03-a-t-06-dominio-puro` | `feat` |
| HU-03 | CA-03.2 ISBN único entre activos | T-02, T-08 | Índice parcial `WHERE deleted_at IS NULL`; repetido → 409 `ISBN_TAKEN` | `feature/T-02-migracion-inicial` | `feat` |
| HU-03 | CA-03.3 total ≥ 1, nunca < 0 | T-04, T-08, T-18 | `ck_book_total_positive` y `ck_book_available_range` en `library:schema-report` | `feature/T-18-checks-al-motor` | `feat` |
| HU-03 | CA-03.4 portada y `cover_key` nula | T-15 | PNG de 2 MB ok; 3 MB → 422; sin portada `null`, nunca `""` | `feature/T-15-portadas-disco` | `test` |
| HU-03 | CA-03.5 baja lógica y 409 | T-04, T-08 | Con préstamo abierto → 409 `BOOK_HAS_OPEN_LOANS`; dado de baja → 404 | `feature/T-08-casos-de-uso-libros` | `feat` |
| HU-04 | CA-04.1 búsqueda y paginación | T-08 | `SearchBooksServiceTest`: `q`, categoría, activos, orden, `size` ≤ 50 | `feature/T-08-casos-de-uso-libros` | `feat` |
| HU-04 | CA-04.2 total de resultados | T-08 | `total` presente en el sobre de paginación | `feature/T-08-casos-de-uso-libros` | `test` |
| HU-04 | CA-04.3 inexistente o dado de baja → 404 | T-08, T-16 | Filtro global `deleted_at IS NULL` (ADR-003) | `feature/T-16-http-contrato-y-errores` | `test` |
| HU-05 | CA-05.1 documento único | T-08 | Repetido → 409 `DOCUMENT_TAKEN` | `feature/T-08-casos-de-uso-libros` | `test` |
| HU-05 | CA-05.2 lista y búsqueda | T-08 | `SearchMembersServiceTest` | `feature/T-08-casos-de-uso-libros` | `feat` |
| HU-05 | CA-05.3 los socios no se eliminan | T-16 | No existe ruta `DELETE /members/{id}` | `feature/T-16-http-contrato-y-errores` | `test` |
| **HU-06** | **CA-06.3/RNF-2 sin ejemplares** | **T-09, T-13** | **`LastCopyRaceTest`: un 201 y un 409** | **`feature/T-13-persistencia-prestamos-y-concurrencia`** | **`fix(db)`** |
| HU-06 | CA-06.1 y CA-06.2 de 1 a 3 líneas | T-05, T-09 | 4 elementos → 422; libro repetido → 422 `DUPLICATE_BOOK_IN_LOAN` | `feature/T-09-caso-de-uso-prestar` | `test` |
| HU-06 | CA-06.4 máximo 3 | T-05, T-09 | 422 `MEMBER_LIMIT_EXCEEDED` con el conteo actual | `feature/T-09-caso-de-uso-prestar` | `test` |
| HU-06 | CA-06.5 socio con vencidos | T-05, T-09 | 409 `MEMBER_HAS_OVERDUE`, evaluado **antes** del límite ([INC-12](consistencia.md)) | `feature/T-09-caso-de-uso-prestar` | `test` |
| HU-06 | CA-06.6 vencimiento y congelado | T-05, T-09 | `due_at == loaned_at + 14 d`; título, ISBN y categoría congelados | `feature/T-05-dominio-loan` | `feat` |
| HU-06 | CA-06.7 no se edita ni borra | T-16 | No existen `PATCH /loans/{id}` ni `DELETE /loans/{id}` | `feature/T-16-http-contrato-y-errores` | `test` |
| HU-07 | CA-07.1 devolución por línea | T-10 | `ReturnItemServiceTest` | `feature/T-10-devolver-y-vencidos` | `feat` |
| HU-07 | CA-07.2 nunca suma dos veces | T-10, T-13 | CAS sobre `returned_at IS NULL`; 409 `ITEM_ALREADY_RETURNED` | `feature/T-13-persistencia-prestamos-y-concurrencia` | `test` |
| HU-07 | CA-07.3 disponibles ≤ total | T-02 | `ck_book_available_range` en el esquema | `feature/T-02-migracion-inicial` | `feat` |
| HU-07 | CA-07.4 `closed` al cerrar | T-05 | `status` calculado en `Loan` | `feature/T-05-dominio-loan` | `test` |
| HU-08 | CA-08.1 detalle con estado | T-16 | `GET /loans/{id}` incluye `status` calculado | `feature/T-16-http-contrato-y-errores` | `feat` |
| HU-08 | CA-08.2 filtros y orden | T-10 | `from`, `to`, `status`, `member_id`, `loaned_at DESC`, paginado | `feature/T-10-devolver-y-vencidos` | `feat` |
| HU-08 | CA-08.3 el estado se calcula | T-05, T-19 | `library:schema-report` no lista columna `status` ni `days_late` | `feature/T-19-pruebas-de-arquitectura` | `test` |
| HU-09 | CA-09.1 y CA-09.2 vencidos | T-10 | `days_late` calculado, orden por más retraso, roles admin y librarian | `feature/T-10-devolver-y-vencidos` | `feat` |
| **HU-10** | **CA-10.1 y CA-10.4 congelado** | **T-11, T-13** | **`FrozenReportTest`: renombrar no cambia el reporte (ADR-004)** | **`feature/T-11-reporte-mas-prestados`** | **`test`** |
| HU-10 | CA-10.2 rango invertido y vacío | T-11 | `from > to` → 422 `INVALID_DATE_RANGE`; rango vacío → `rows: []` | `feature/T-11-reporte-mas-prestados` | `test` |
| HU-10 | CA-10.3 sin desglose | T-11 | El reporte no expone `member_id` ni `loaned_by_user_id` | `feature/T-11-reporte-mas-prestados` | `test` |
| HU-10 | CA-10.5 solo admin | T-16 | `librarian` → 403 `FORBIDDEN_ROLE` | `feature/T-16-http-contrato-y-errores` | `test` |

## 2. Reglas de negocio

| Regla | Tarea | Evidencia | Rama |
|---|---|---|---|
| RN-1 disponibles en rango | T-04, T-02 | `ck_book_available_range` + pruebas del dominio | `feature/T-02-migracion-inicial` |
| RN-2 14 días y máximo 3 | T-05 | Pruebas del dominio | `feature/T-05-dominio-loan` |
| RN-3 socio con vencidos | T-09 | `LendBooksServiceTest` | `feature/T-09-caso-de-uso-prestar` |
| RN-4 lo congelado no sigue al catálogo | T-11 | `FrozenReportTest` | `feature/T-11-reporte-mas-prestados` |
| RN-5 nada se borra | T-08, T-16 | FK `RESTRICT`, `deleted_at`, sin rutas de borrado | `feature/T-16-http-contrato-y-errores` |
| RN-6 UTC y español | T-02, T-16 | `timestamptz` + `timezone = UTC`; mensajes en español | `feature/T-16-http-contrato-y-errores` |

## 3. Requisitos no funcionales

| RNF | Tarea | Evidencia | Rama |
|---|---|---|---|
| RNF-1 búsqueda < 300 ms con 5 000 libros | T-21 | Latencia p50/p95 con `tool --books=5000`. **Antes de añadir índices**: ver [INC-11](consistencia.md) | `feature/T-21-tool-sembrador` |
| RNF-2 último ejemplar en concurrencia | T-13 | `LastCopyRaceTest` | `feature/T-13-persistencia-prestamos-y-concurrencia` |
| RNF-3 la contraseña nunca viaja ni se loguea | T-14 | `NoSecretLeakTest` + `gitleaks` | `feature/T-19-pruebas-de-arquitectura` |
| RNF-4 cuerpo de error único | T-16 | `ErrorEnvelopeTest` sobre las 20 rutas | `feature/T-16-http-contrato-y-errores` |
| RNF-5 un comando y base vacía | T-20 | `docker compose up`; la base sin tablas hasta migrar | `feature/T-20-infra-docker-compose` |

## 4. Constitución

| Artículo | Tarea | Evidencia | Rama |
|---|---|---|---|
| I · El dominio no depende de nada | T-19 | `NoExternalDependenciesTest` falla ante `Illuminate\`, `Carbon\` o `Adapters` | `feature/T-19-pruebas-de-arquitectura` |
| II · El núcleo declara sus puertos | T-19 | `deptrac` + escaneo de firmas de `app/Application/Ports` | `feature/T-19-pruebas-de-arquitectura` |
| III · Un único punto de composición | T-17, T-19 | `SingleCompositionRootTest` falla ante un `new` de adaptador fuera de `PortBindings` | `feature/T-17-composicion-y-admin-inicial` |
| IV · SOLID no es decorativo | T-07..T-11 | Un test por servicio; fakes que sustituyen al adaptador real (LSP) | `feature/T-09-caso-de-uso-prestar` |
| V · El esquema es de las migraciones | T-02, T-20 | `infra` no contiene `CREATE TABLE` | `feature/T-20-infra-docker-compose` |
| VI · La base impone lo que puede | T-02, T-18 | `library:schema-report` contra las consultas de §10 | `feature/T-18-checks-al-motor` |
| VII · Lo derivado se calcula | T-05, T-19 | Sin columna `status` ni `days_late` | `feature/T-19-pruebas-de-arquitectura` |
| VIII · Lo congelado no sigue al catálogo | T-11 | `FrozenReportTest` | `feature/T-11-reporte-mas-prestados` |
| IX · Ningún secreto en el repo ni en un log | T-14, T-19 | `gitleaks` en CI; `git diff --cached` nunca lista `.env` | `feature/T-19-pruebas-de-arquitectura` |
| X · Gana el motor | — | Procedimiento en [`consistencia.md`](consistencia.md) §5 | — |
| XI · Idioma | T-16 | Columnas y rutas en inglés; mensajes en español | `feature/T-16-http-contrato-y-errores` |
| XII · Nada sin evidencia | Todas | Cada tarea se cierra con un comando o un test que pasa | — |

## 5. Los seis repositorios

| Repo | Ramas | Tipo de commit dominante |
|---|---|---|
| `docs` | `docs/…` | `docs` |
| `api` | `feature/T-01…` a `feature/T-19…` | `feat`, `fix`, `test` |
| `infra` | `feature/T-20-infra-docker-compose` | `build` |
| `tool` | `feature/T-21-tool-sembrador` | `feat` |
| `app` | `feature/T-22-app-react` | `feat` |
| `page` | `feature/T-23-page-estatica` | `feat` |
