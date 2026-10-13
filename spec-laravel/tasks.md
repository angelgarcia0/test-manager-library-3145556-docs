# Tareas — Manager Library (Laravel)

> **Fuente normativa:** [`../spec-python/tasks.md`](../spec-python/tasks.md). Las tareas son **las
> mismas**; lo que cambia es la tecnología de la evidencia.

Cada tarea termina cuando su **evidencia** existe y se puede ejecutar (artículo XII). Estado inicial
de todas: **pendiente**.

| # | Tarea | Depende de | Evidencia |
|---|---|---|---|
| T-01 | Esqueleto de Laravel y `GET /health` | — | `curl -i /api/v1/health` responde `{"status":"ok"}`; el contenedor arranca con un comando |
| T-02 | Migración inicial: seis tablas, FK, índices únicos y las cinco categorías sembradas | T-01 | `php artisan library:schema-report` devuelve lo documentado; ninguna columna con `DEFAULT`; un test falla si aparece una |
| T-03 | Dominio: `Isbn`, `Quantity`, `DateRange` (objetos de valor) | T-01 | Pruebas unitarias: ISBN de 13 dígitos, cantidad > 0, fin ≥ inicio |
| T-04 | Dominio: `Book` (`withdraw`, `restock`, baja lógica) | T-03 | Pruebas: nunca < 0, nunca > total |
| T-05 | Dominio: `Loan` (`addItem`, `returnItem`, estado calculado, 14 días, máx. 3) | T-04 | Pruebas con un instante fijo: `open`, `overdue`, `closed` |
| T-06 | Dominio: `Member` y `User` (normalización de nombre, rol válido) | T-03 | `"Ana "` queda `ana`; rol `admin` por API rechazado |
| T-07 | Puertos y casos de uso de **autenticación** (`LoginService`, `CreateLibrarianService`) con fakes | T-06 | HU-01 y HU-02 pasan contra fakes |
| T-08 | Puertos y casos de uso de **libros** (crear, buscar, editar, baja, portada) con fakes | T-04 | HU-03 y HU-04 pasan contra fakes |
| T-09 | Caso de uso **prestar** (`LendBooksService`) con fakes | T-05, T-08 | HU-06 completa: límites, vencidos, sin ejemplares; orden 409 antes que 422 |
| T-10 | Caso de uso **devolver** y **vencidos** | T-09 | HU-07 y HU-09; devolver dos veces → `ITEM_ALREADY_RETURNED` |
| T-11 | Caso de uso **reporte de más prestados** y su puerto de lectura | T-09 | HU-10; el título viejo se conserva tras renombrar (ADR-004) |
| T-12 | Adaptador de persistencia: repositorios de libro, socio, usuario | T-02, T-08 | Pruebas de integración contra PostgreSQL real |
| T-13 | Adaptador de persistencia: repositorio de préstamos + compare-and-swap | T-12, T-09 | Dos préstamos del último ejemplar: uno 201, otro 409 (RNF-2). [ADR-005](adr/adr-005-compare-and-swap.md) |
| T-14 | Adaptador de seguridad: hash Argon2id y token JWT | T-07 | La clave nunca aparece en logs ni respuestas (RNF-3); el esquema sigue con seis tablas |
| T-15 | Adaptador de portadas (disco) | T-08 | Subir PNG de 2 MB ok; 3 MB → 422; sin portada → `cover_key` nulo, nunca `""` |
| T-16 | Adaptador HTTP: rutas del contrato, cuerpo de error único, autorización por rol | T-10, T-11, T-14, T-15 | Prueba de contrato recorre todas las rutas y todos los códigos de §2. [ADR-008](adr/adr-008-cuerpo-de-error-unico.md) |
| T-17 | Composición y arranque (admin inicial desde entorno) | T-16 | El sistema arranca con la base vacía y se puede iniciar sesión como admin |
| T-18 | Bajar a la base las invariantes **solo dominio**: `ck_book_total_positive`, `ck_user_role` | T-12 | Un `INSERT` por `psql` que las viole falla; `library:schema-report` las lista |
| T-19 | Pruebas de arquitectura (artículos I, II y III) | T-17 | Una regla de `deptrac` y un test en `tests/Architecture` pasan y fallan si se introduce una importación prohibida |
| T-20 | `infra`: `docker compose up` levanta motor vacío, API y frontend | T-17 | Un comando; la base arranca sin tablas hasta correr las migraciones |
| T-21 | `tool`: sembrador de demostración **por la API** (≥ 40 libros, 10 socios, 15 préstamos, 3 vencidos) | T-17 | Tras correrlo, `/loans/overdue` devuelve 3 líneas y el reporte tiene filas. [ADR-007](adr/adr-007-reloj-como-puerto.md) |
| T-22 | `app`: login, catálogo, socios, prestar, devolver, vencidos, reporte | T-16 | Recorrido manual de punta a punta en la feria |
| T-23 | `page`: sitio público estático de presentación | — | Se abre sin la API; correcto en móvil y escritorio |

## Qué cambió respecto a las tareas del spec de referencia

Ninguna tarea desaparece ni se parte. Estos son los ajustes de **evidencia** que la traducción
obliga:

| Tarea | Evidencia del spec de referencia | Evidencia en Laravel | Por qué |
|---|---|---|---|
| T-02 | «Las consultas de §10 devuelven lo documentado» | `php artisan library:schema-report`, que ejecuta **literalmente** esas consultas | El artículo VI pide evidencia ejecutable, y en PHP la consulta se emite desde un comando |
| T-05 | «Pruebas con un `Clock` falso» | Pruebas de dominio con un `DateTimeImmutable` fijo; las de aplicación con un `FakeClock` | El dominio no puede importar el puerto ([INC-13](consistencia.md)) |
| T-13 | «testigo de versión» | Pruebas de concurrencia sobre compare-and-swap | No hay columna de versión ([ADR-005](adr/adr-005-compare-and-swap.md)) |
| T-15 | «3 MB → 422» | Ídem, más `upload_max_filesize=8M` en `php.ini` | Si PHP limita antes, respondería 413 ([INC-07](consistencia.md)) |
| T-19 | «un contrato de `import-linter`» | `deptrac` **y** un test propio en `tests/Architecture` | `import-linter` no existe para PHP |
| T-21 | «3 vencidos por la API» | Dos fases del sembrador con `CLOCK_OFFSET_DAYS` | [ADR-007](adr/adr-007-reloj-como-puerto.md) |

## Definición de terminado de la prueba

Quien llegue a T-17 con evidencia tiene el **núcleo** hecho. T-18 a T-23 son lo que separa un
trabajo bueno de uno sobresaliente. Se evalúa el desempeño con SDD, no la cantidad de tareas: una
tarea con evidencia vale más que tres a medias.

Una tarea está terminada cuando su evidencia **se ejecuta y pasa**, no cuando «ya está
programada».
