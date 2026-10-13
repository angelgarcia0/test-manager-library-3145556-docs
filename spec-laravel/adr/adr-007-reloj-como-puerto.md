# ADR-007 · El reloj es un puerto, y eso permite sembrar vencidos

- **Estado:** aceptada
- **Alcance:** traducción a Laravel de la decisión D-05 de [`plan.md`](../plan.md)
- **Resuelve:** [INC-04](../consistencia.md) de [`consistencia.md`](../consistencia.md)

## Contexto

`tasks.md` T-21 exige que el sembrador de `tool` —que, según `data-model.md` §9, usa la **API** y
«nunca SQL directo»— deje **3 líneas vencidas** en `/loans/overdue`.

Pero CA-06.6 fija `due_at = loaned_at + 14 días`, con `loaned_at` igual al instante actual, y
`POST /loans` no admite ningún parámetro de fecha. Un cliente que solo habla HTTP **no puede
producir ni un solo préstamo vencido**: tendría que esperar 14 días.

El spec no resuelve el choque, pero sí ofrece la herramienta: `plan.md` D-05 dice que «el reloj es
un puerto (`Clock`)» con la razón explícita de «**probar vencimientos sin esperar 14 días**», y el
artículo II bendice el puerto sin que el núcleo sepa quién lo implementa.

## Decisión

El puerto `Clock` se implementa con **dos adaptadores** en `app/Adapters/Outbound/`:

| Adaptador | Cuándo | Comportamiento |
|---|---|---|
| `SystemClock` | Por defecto, siempre en producción | `new DateTimeImmutable('now', new DateTimeZone('UTC'))` |
| `ShiftableClock` | Solo si existe `CLOCK_OFFSET_DAYS` | Devuelve `ahora + N días` |

`PortBindings` selecciona `ShiftableClock` únicamente cuando la variable está definida; sin ella, el
sistema es indistinguible de producción. El sembrador de `tool` ejecuta sus dos fases con la API
desplazada:

| Fase | `CLOCK_OFFSET_DAYS` | Efecto |
|---|---|---|
| 1 | `+20` | Los préstamos creados **nacen vencidos**, con 6 días de retraso |
| 2 | `0` (o ausente) | Préstamos vigentes, devoluciones variadas y reporte con datos |

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| SQL directo desde `tool` | `data-model.md` §9 lo prohíbe expresamente |
| Un parámetro `loaned_at` o `due_at` en `POST /loans` | **Alteraría el contrato HTTP** y abriría la puerta a falsificar el histórico (RN-5) |
| Esperar 14 días | Inaceptable: el plazo de entrega es el mismo día |
| Editar `due_at` por `psql` | Prohibido por la misma razón que el SQL directo en `tool` |
| Congelar el reloj dentro del dominio | El artículo I prohíbe que el dominio dependa de una abstracción externa; ver [INC-13](../consistencia.md) |

## Consecuencias

- **No se toca el modelo de datos, ni el dominio, ni el contrato HTTP.** Solo cambia qué clase
  cumple un puerto que el artículo II ya obligaba a declarar.
- El desplazamiento vive en la **capa de adaptador**, nunca en un caso de uso: ningún servicio
  conoce `CLOCK_OFFSET_DAYS`.
- Con la variable definida, el `CHECK` y las reglas de negocio siguen aplicando igual: un socio con
  vencidos no puede pedir más (RN-3) y el reporte se calcula sobre `loaned_at` reales.
- Por ser una costura explícita, el mismo puerto sirve para probar T-05 (`open`, `overdue`,
  `closed`) con un `FakeClock`, sin esperar y sin tocar la base.

## Cómo se verifica

T-21: tras ejecutar el sembrador, `GET /loans/overdue` devuelve **exactamente 3** líneas y
`GET /reports/most-borrowed` devuelve filas.

Complemento, en el repo `api`: con `CLOCK_OFFSET_DAYS` **ausente**, un préstamo recién creado figura
como `open` y no aparece en `/loans/overdue`.
