# ADR-006 · El token es un JWT sin estado

- **Estado:** aceptada
- **Alcance:** traducción a Laravel de la decisión D-11 de [`plan.md`](../plan.md)
- **Resuelve:** [INC-03](../consistencia.md) de [`consistencia.md`](../consistencia.md)

## Contexto

`plan.md` D-11: «Autenticación — **JWT HS256**, 8 horas, clave en `JWT_SECRET`. Simple y sin
estado». `api-contract.md` §4: «Token — **JWT firmado, 8 horas, con `sub` y `role`**. Sin datos
personales dentro».

Pero las dos opciones que Laravel ofrece para JWT **crean esquema**:

| Paquete | Tabla que crea | Por qué no vale |
|---|---|---|
| `laravel/sanctum` | `personal_access_tokens` | El token es opaco y **almacenado**: contradice «sin estado». Además registra `last_used_at`, una columna de auditoría que `data-model.md` §8 prohíbe |
| `tymon/jwt-auth` | `token_in_blacklistable_tokens` | La séptima tabla existe aunque la lista negra esté apagada: la migración está en el repositorio |
| `firebase/php-jwt` | ninguna | Puro, HS256, sin estado, sin base de datos |

`data-model.md` §2 es tajante: «Seis entidades, seis tablas. No hay tabla de reporte, de multas ni
de auditoría», y §10 verifica el catálogo de índices y restricciones del esquema `library`.

## Decisión

**`firebase/php-jwt`.** Sin migraciones de terceros, sin tabla de tokens, sin lista negra.

| Aspecto | Valor |
|---|---|
| Algoritmo | HS256 |
| Secreto | `JWT_SECRET` del entorno (artículo IX: nunca en el repositorio) |
| Vigencia | 8 h → `expires_in: 28800` (CA-01.1) |
| Claims | `sub` (uuid del usuario) y `role` — **nada más** |
| Errores de firma o expiración | 401 `INVALID_CREDENTIALS`, con el mismo mensaje que unas credenciales inválidas (CA-01.2) |

La verificación de firma, expiración y presencia de `role` ocurre en el middleware `Authenticate`,
nunca en un controlador.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| `laravel/sanctum` | Token con estado + tabla `personal_access_tokens` |
| `tymon/jwt-auth` | Tabla `token_in_blacklistable_tokens`; además arrastra su propio modelo de usuario, que aquí es `library."user"` |
| Sesiones de Laravel | El contrato exige `Authorization: Bearer <token>` y no hay cookie ni dominio de sesión |
| Guardar el token en caché | `architecture.md` §6: «sin caché». Añade además infraestructura no solicitada |

## Consecuencias

- **No se ejecuta ninguna migración de paquete de autenticación.** Tras `migrate:fresh --seed` el
  esquema `library` tiene exactamente seis tablas, y así lo comprueba
  `php artisan library:schema-report` (las consultas de `data-model.md` §10).
- El token es autocontenido: no hay revocación. No es una carencia, porque **la API no permite
  cambiar el rol de un usuario** y solo el despliegue crea el `admin`, así que durante los 8 h de
  vigencia el rol del token sigue siendo el vrai.
- CA-01.2 exige que el mensaje no diga si falló el usuario o la clave: con usuario inexistente se
  ejecuta igualmente una verificación de hash de mentira, para no filtrar por tiempo de respuesta.

## Cómo se verifica

- Un test afirma que `information_schema.tables` del esquema `library` devuelve **seis** filas.
- Un test recorre las 8 h de vigencia con un reloj controlado: token válido → 200; vencido → 401.
- Un test afirma que **ningún** claim del token contiene `password`, `password_hash`, nombre ni
  documento (artículo IX).
- RNF-3: ninguna respuesta ni log contiene `password` ni `password_hash`.
