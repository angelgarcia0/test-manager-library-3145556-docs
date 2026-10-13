# ADR-008 · Un único manejador produce el único cuerpo de error

- **Estado:** aceptada
- **Alcance:** traducción a Laravel de RNF-4 y del artículo II de `api-contract.md`
- **Resuelve:** [INC-08](../consistencia.md) y [INC-06](../consistencia.md) de [`consistencia.md`](../consistencia.md)

## Contexto

`spec.md` RNF-4: «Todo error HTTP usa el mismo cuerpo (ver contrato)». El cuerpo es único:

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`api-contract.md` lo cierra: «`details` puede ir vacío. **Nunca** incluye trazas, SQL, rutas del
servidor ni la contraseña».

Pero la tabla §2 lista **catorce códigos de negocio** y ninguno cubre lo que Laravel emite por su
cuenta: una `ValidationException` con su propio sobre, un `NotFoundHttpException` de ruta, un
`MethodNotAllowedHttpException`, o un `QueryException` whose message contiene el SQL que falló.

## Decisión

**Un solo manejador global** renderiza el mismo sobre para **toda** respuesta de error, incluidos
los que Laravel genera internamente. Tabla de mapeo, completa y explícita:

| Origen | HTTP | `code` |
|---|---|---|
| `ValidationException` / FormRequest | 422 | `VALIDATION_ERROR` |
| Sin token, token inválido o vencido | 401 | `INVALID_CREDENTIALS` |
| Credenciales inválidas en el login | 401 | `INVALID_CREDENTIALS` |
| Rol insuficiente | 403 | `FORBIDDEN_ROLE` |
| Modelo no encontrado o dado de baja | 404 | `NOT_FOUND` |
| `QueryException` SQLSTATE `23505` (único) | 409 | mapeado por columna, abajo |
| `QueryException` SQLSTATE `23514` (`CHECK` violado) | 500 | `INTERNAL_ERROR` |
| Ruta inexistente | 404 | `NOT_FOUND` |
| Método no permitido | 405 | `METHOD_NOT_ALLOWED` |
| Cualquier otra excepción | 500 | `INTERNAL_ERROR` |

Mapeo de las violaciones de unicidad a códigos de negocio:

| Restricción única | `code` |
|---|---|
| `user.username` | `USERNAME_TAKEN` |
| `book.isbn` (índice parcial de activos) | `ISBN_TAKEN` |
| `member.document_number` | `DOCUMENT_TAKEN` |
| `loan_item (loan_id, book_id)` | `DUPLICATE_BOOK_IN_LOAN` |

**Regla dura:** el mensaje del driver **nunca** sale. El `QueryException` se registra en el log con
un identificador de correlación que sí se devuelve, y el cliente recibe un mensaje en español
(artículo XI) sin SQL ni trazas.

**`CHECK` violado es un defecto, no un caso de negocio** (ADR-002). Se responde 500 con el sobre
único, se registra con correlación y se investiga; nunca se disfraza de 409.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Dejar el `Handler` de Laravel por defecto | Devuelve varios formatos distintos: incumple RNF-4 |
| Excepciones por endpoint | Un endpoint olvidado rompe el contrato; elmanejo único no se puede olvidar |
| Devolver el `code` del error de framework en crudo | Filtraría nombres de clases y de tablas |
| Quitar `throttle` y aceptar el 429 | Un 429 es un estado **inexistente** en `api-contract.md` §1 |

## Consecuencias

- El grupo de rutas `api` se declara **sin `throttle`**: el `throttle:api` por defecto emitiría un
  429 que el contrato no define. No se añade rate limiting, porque ningún requisito lo pide y
  `architecture.md` §6 es explícito: «se añaden si un requisito lo pide, no antes».
- `INTERNAL_ERROR` y `METHOD_NOT_ALLOWED` son las **dos** adiciones de vocabulario que la
  Constitución exige para poder cumplir RNF-4. No cambian alcance, negocio ni contrato de negocio.
- `details` se construye con el campo y el problema en español; nunca con la excepción de origen.
- Los códigos de negocio del §2 se emiten **desde el dominio y la aplicación**, nunca desde el
  adaptador HTTP: el adaptador solo traduce (artículo II de `architecture.md` §5).

## Cómo se verifica

T-16 / RNF-4: un test recorre las veinte rutas del contrato y los catorce códigos de §2, y afirma
que **toda** respuesta con estado ≥ 400 —incluidas ruta inexistente, método no permitido, token
ausente y rol insuficiente— tiene exactamente las claves `code`, `message` y `details`, que
`details` es un arreglo, y que ningún cuerpo contiene `password`, `password_hash`, `stack trace`,
`select ` ni el nombre de una tabla con el prefijo `library.`
