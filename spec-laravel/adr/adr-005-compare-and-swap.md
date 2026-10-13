# ADR-005 · El testigo de versión es `available_copies`

- **Estado:** aceptada
- **Alcance:** traducción a Laravel de [ADR-002](../spec-python/adr/adr-002-concurrencia-optimista.md)
- **Resuelve:** [INC-01](../consistencia.md) de [`consistencia.md`](../consistencia.md)

## Contexto

ADR-002 (aceptada) y `plan.md` D-04 exigen concurrencia **optimista** con «testigo de versión por
libro»: al guardar, si la versión cambió, la operación falla con 409 `BOOK_NOT_AVAILABLE`. El
diagrama de secuencia de `architecture.md` §4 muestra `L-->>S: ok o conflicto de versión`.

Pero `data-model.md` §3 fija **exactamente nueve columnas** en `book` y ninguna es un testigo:

```
id, title, author, isbn, category_id, total_copies, available_copies, cover_key, deleted_at
```

§3 prohíbe además cualquier `DEFAULT`, y §10 exige que `information_schema.columns` devuelva
«las columnas de §3». **No se puede añadir una columna `version`.**

## Decisión

El testigo de versión **es el propio `available_copies`**, mediante *compare-and-swap*. El valor
leído antes de escribir es la versión; si cambió, la actualización no afecta filas.

```sql
UPDATE library.book
   SET available_copies = :nuevo
 WHERE id = :id
   AND available_copies = :esperado
   AND available_copies > 0;
```

En PostgreSQL con `READ COMMITTED`, la condición del `WHERE` se **reevalúa después** de obtener el
bloqueo de fila: si otra transacción escribió antes, la fila ya no cumple `available_copies =
:esperado`, la actualización afecta cero filas y se reporta el conflicto. La semántica es CAS real.

**El valor esperado vive en el adaptador, no en el dominio.** `BookRepository` conserva el
`available_copies` leído en el `SELECT` que cargó el agregado; el objeto `Book` de dominio queda
mudo de persistencia, como exige el artículo I.

**Las devoluciones usan el mismo mecanismo sobre `returned_at`:**

```sql
UPDATE library.loan_item
   SET returned_at = now()
 WHERE id = :id
   AND returned_at IS NULL;
```

Cero filas significa que la línea ya estaba devuelta → 409 `ITEM_ALREADY_RETURNED` (CA-07.2), y
garantiza el «nunca suma dos veces» sin depender solo del `CHECK`.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Añadir `book.version integer` | **Prohíbido**: altera el modelo de datos y rompe `data-model.md` §3 y §10 |
| `lockForUpdate()` de Laravel | Es el bloqueo pesimista que ADR-002 descarta: serializa toda la biblioteca por un caso raro |
| `increment()` / `decrement()` de Eloquent | `SET x = x - 1` no detecta el conflicto: dos Librarianías prestarían el **último** ejemplar. Es la forma más sutil de romper RNF-2 |
| Bloqueo en memoria (contador) | No sobrevive a más de un proceso, como ya dice ADR-002 |

## Consecuencias

- `ck_book_available_range` (`0 <= available_copies <= total_copies`) sigue intacto y sigue siendo
  la **última barrera**: si salta, algo escribió fuera del adaptador y es un defecto, no un caso de
  negocio.
- El decremento y la creación de la línea de préstamo ocurren **en una sola transacción** (CA-06.3):
  si falta un ejemplar, no se registra nada.
- ADR-002 autoriza un reintento único: releer el libro y reintentar **una** vez solo si sigue
  habiendo ejemplar.
- El repositorio debe exponer el resultado como un valor de retorno que el caso de uso traduzca a
  `BookNotAvailable`; el `CHECK` violado nunca debe llegar al cliente como un 409 de negocio.

## Cómo se verifica

T-13 / RNF-2: dos peticiones `POST /loans` **simultáneas** sobre un libro con
`available_copies = 1` → exactamente una recibe **201** y la otra **409
`BOOK_NOT_AVAILABLE`**; al final `available_copies = 0` y existe una sola línea de préstamo.

Complemento: dos devoluciones simultáneas de la misma línea → un 200, un 409
`ITEM_ALREADY_RETURNED`, y `available_copies == total_copies` (nunca superado, CA-07.3).
