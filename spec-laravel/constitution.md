# Constitución — Manager Library (Laravel)

> **Fuente normativa:** [`../spec-python/constitution.md`](../spec-python/constitution.md). Los doce
> artículos son **los mismos** y son innegociables. Solo cambian las rutas, las herramientas y la
> forma de verificarlos.

Principios innegociables del proyecto. No son estilo ni preferencia: cada artículo trae **cómo se
verifica**, porque un principio que nadie puede comprobar es una intención. Un cambio que viole un
artículo se rechaza sin discutir el resto.

---

## Artículo I — El dominio no depende de nada

`app/Domain/` no importa ningún otro módulo del proyecto ni librerías de terceros: solo PHP
estándar (`final readonly class`, `DateTimeImmutable`, `InvalidArgumentException`); ni Eloquent, ni
Illuminate, ni Facades, ni **`Carbon`**.

> En PHP la restricción es más fuerte que en Python: `Carbon` es una librería de terceros, tan
> prohibida aquí como SQLAlchemy o pydantic lo están en el spec de referencia. El dominio habla
> `DateTimeImmutable`; ver la [nota de traducción](#nota-de-traducción--el-dominio-habla-datetimeimmutable).

**Por qué:** es lo único que permite probar las invariantes sin base de datos, sin servidor y sin
contenedor, y cambiar de ORM sin tocar reglas.

**Verificación:** una regla de `deptrac` (`Domain` no depende de nadie) y un test en
`tests/Architecture/NoExternalDependenciesTest.php` que recorre los `use` de `app/Domain` y falla
ante `Illuminate\`, `Carbon\`, `Laravel\`, `Ramsey\` o cualquier clase de `App\Adapters`.

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

La capa de aplicación solo conoce el dominio. Todo lo que necesita del exterior (repositorios,
reloj, hash de claves, almacenamiento de portadas, generación de tokens) lo expresa como un **puerto
que ella misma define**. Ningún tipo de framework o de ORM aparece en la firma de un puerto.

**Verificación:** una regla de `deptrac` impide que `Application` dependa de `Adapters`, y un test
recorre las firmas de `app/Application/Ports` buscando `Illuminate\`, `Illuminate\Database\Eloquent`
o `Psr\Http`. Los tests de aplicación corren con fakes y sin referenciar `Adapters`.

> El puerto `Clock` es la excepción que resuelve la tensión de [INC-13](consistencia.md): vive en
> la aplicación (este artículo y el IV) y **quien lo consume lo invoca y le pasa el
> `DateTimeImmutable`** al dominio, para que el dominio no importe nada externo.

## Artículo III — Un único punto de composición

Un solo archivo (`app/Bootstrap/Composition/PortBindings.php`) es donde un puerto se encuentra con
su adaptador. Fuera de él no se instancia infraestructura.

**Verificación:** un test en `tests/Architecture/SingleCompositionRootTest.php` que recorre
`app/Adapters` y falla ante cualquier `new` de una clase concreta de adaptador fuera de
`PortBindings`. Todos los puertos se enlazan de forma **explícita** —no por resolución automática
del contenedor— para que el artículo sea auditable.

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| **SRP** | Un servicio de aplicación = un caso de uso. `LendBooksService` presta y no hace nada más | Un servicio con dos razones para cambiar se parte |
| **OCP** | Cambiar el almacenamiento de portadas (disco → S3) no toca dominio ni aplicación | El adaptador nuevo solo añade una línea en la composición |
| **LSP** | Los fakes de test sustituyen al adaptador real sin cambiar expectativas | El mismo test pasa contra el fake y el real |
| **ISP** | Puertos pequeños: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | Ningún implementador deja métodos sin usar |
| **DIP** | El puerto lo define quien lo consume | Los puertos viven en `app/Application/Ports/Outbound` |

## Artículo V — El esquema es propiedad de las migraciones

El repositorio `infra` levanta un motor **vacío**. Todo el DDL (tablas, índices, restricciones,
esquema `library` y semilla) vive en las migraciones de la aplicación. Ver
[ADR-001](../spec-python/adr/adr-001-propiedad-del-esquema.md).

**Verificación:** `infra` no contiene `CREATE TABLE`; ni `infra/db/init/` ni volúmenes de
inicialización de Postgres. `CREATE SCHEMA IF NOT EXISTS library` también va en la migración.

## Artículo VI — Las invariantes que la base puede imponer, la base las impone

Una regla que solo vive en código protege a la aplicación, no a los datos. Toda invariante
expresable en el motor (`CHECK`, `UNIQUE`, FK) se declara ahí. Lo que no está aplicado se marca
**pendiente** en [`../spec-python/data-model.md`](../spec-python/data-model.md); no se promete.

**Verificación:** el comando `php artisan library:schema-report` ejecuta **literalmente** las tres
consultas de `data-model.md` §10 y su salida se compara con lo que declara el documento.

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan. No hay
columna para ellos.

**Verificación:** `data-model.md` no lista ninguna columna `status` ni `days_late`, y
`library:schema-report` confirma que la base no tiene ninguna.

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo guarda título, ISBN y categoría del instante del préstamo. Ningún reporte los
lee del catálogo vivo.

**Verificación:** una prueba que renombra un libro después de prestarlo y comprueba que el reporte
conserva el título viejo ([ADR-004](../spec-python/adr/adr-004-reporte-agregado-y-congelado.md)).

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ni contraseñas, ni hashes precalculados, ni la clave del token. Todo secreto viene del entorno. El
`.env` real está en `.gitignore`; solo se versiona `.env.example`.

**Verificación:** `gitleaks` en CI; ninguna respuesta HTTP incluye `password` ni `password_hash`;
`git diff --cached --name-only` nunca lista `.env`; ninguna línea de log incluye el cuerpo de
`POST /auth/login` ni de `POST /users`.

## Artículo X — Gana el motor

Si el documento de datos y la base discrepan, el documento está roto y se corrige el documento. En
este repo la corrección se propone como un cambio explícito y aislado, nunca como un ajuste
silencioso: ver las [preguntas abiertas](consistencia.md).

## Artículo XI — Idioma

Código, columnas y rutas en inglés. Documentación y mensajes al usuario en español. En Laravel esto
incluye los mensajes de `validation.php` y el cuerpo de error del contrato, que van en español,
mientras que las claves de `lang/` y los nombres de los mensajes de error de negocio van en inglés.

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) está terminada cuando su **evidencia** existe y se puede
ejecutar, no cuando «ya está programada». Toda evidencia de este proyecto es un comando o un test.

---

## Nota de traducción — El dominio habla `DateTimeImmutable`

Esto **no es un artículo nuevo**: es la condición que hace posible el artículo I en PHP, y su
redacción original ya lo cubre.

En Python el artículo I nombra las librerías prohibidas por nombre (`SQLAlchemy`, `FastAPI`,
`pydantic`). En PHP el equivalente es `Illuminate\`, los Facades y **`Carbon`**: `Carbon` es una
librería de terceros, tan prohibida en `app/Domain` como lo están en el spec de referencia. El
dominio usa `DateTimeImmutable` de la biblioteca estándar y los adaptadores convierten en la
frontera. **El artículo I no se relaja: se aplica.**

**Verificación:** el test `NoExternalDependenciesTest` falla si aparece `Carbon` en `app/Domain`.
