# Traducción del spec a React + Laravel

> **Esta carpeta no es el spec.** El spec normativo e inmutable del producto es
> [`../spec-python/`](../spec-python/), equivalente a [`../spec-.net/`](../spec-.net/): mismo
> alcance, mismas historias, mismas reglas, mismo contrato HTTP y mismo modelo de datos.
>
> El enunciado de la prueba se entrega en **PHP con Laravel** y **React**, no en Python ni en
> .NET (ver [`../README.md`](../README.md)). Este documento traduce **únicamente las decisiones de
> tecnología**. No reescribe el negocio, no altera el contrato HTTP y no amplía el modelo de datos.

## Qué contiene esta carpeta

| Archivo | Qué traduce | Fuente normativa |
|---|---|---|
| [`constitution.md`](constitution.md) | Los doce artículos: rutas, herramientas y forma de verificarlos en Laravel | [`../spec-python/constitution.md`](../spec-python/constitution.md) |
| [`plan.md`](plan.md) | Las decisiones D-01..D-12 en Laravel | [`../spec-python/plan.md`](../spec-python/plan.md) |
| [`architecture.md`](architecture.md) | La arquitectura hexagonal sobre el esqueleto de Laravel | [`../spec-python/architecture.md`](../spec-python/architecture.md) |
| [`arquitectura-panoramica.md`](arquitectura-panoramica.md) | El sistema en una página, con el backend en Laravel | [`../spec-python/arquitectura-panoramica.md`](../spec-python/arquitectura-panoramica.md) |
| [`tasks.md`](tasks.md) | T-01..T-23 y la evidencia ejecutable de cada una | [`../spec-python/tasks.md`](../spec-python/tasks.md) |
| [`consistencia.md`](consistencia.md) | Incoherencias halladas al traducir, y mitigación aplicada | — (aporte propio) |
| [`matriz-trazabilidad.md`](matriz-trazabilidad.md) | Requisito → tarea → evidencia → rama → commit | — (aporte propio) |
| [`adr/`](adr/) | ADR-005..ADR-008: decisiones que solo existen en esta traducción | [`../spec-python/adr/`](../spec-python/adr/) |

### Las cuatro ADR de esta traducción

Cada una resuelve una incoherencia que aparece **al traducir**, sin tocar el alcance, el contrato
HTTP ni el modelo de datos. Las ADR-001..004 del spec se respetan tal cual.

| ADR | Decisión | Resuelve |
|---|---|---|
| [ADR-005](adr/adr-005-compare-and-swap.md) | El testigo de versión es `available_copies` (compare-and-swap) | [INC-01](consistencia.md) |
| [ADR-006](adr/adr-006-jwt-sin-estado.md) | El token es un JWT sin estado con `firebase/php-jwt` | [INC-03](consistencia.md) |
| [ADR-007](adr/adr-007-reloj-como-puerto.md) | El reloj es un puerto, y eso permite sembrar vencidos | [INC-04](consistencia.md) |
| [ADR-008](adr/adr-008-cuerpo-de-error-unico.md) | Un único manejador produce el único cuerpo de error | [INC-06](consistencia.md) y [INC-08](consistencia.md) |

## Los documentos que no se copian

`spec.md`, `data-model.md`, `api-contract.md` y `adr-001`..`adr-004` son **idénticos** en las dos
versiones del spec y describen **el producto**, no la pila. Se referencian, no se replican: una
copia podría divergir y se convertiría en una segunda fuente de verdad.

| Documento | Por qué no se copia |
|---|---|
| [`../spec-python/spec.md`](../spec-python/spec.md) | Historias, criterios, reglas y RNF: independientes del lenguaje |
| [`../spec-python/data-model.md`](../spec-python/data-model.md) | El único sitio donde vive el modelo de datos, y es del producto |
| [`../spec-python/api-contract.md`](../spec-python/api-contract.md) | El contrato HTTP es «independiente del lenguaje» (lo dice su encabezado) |
| [`../spec-python/adr/`](../spec-python/adr/) | ADR-001..004 ya son válidos tal cual en Laravel |

## Regla de trabajo

Ni [`../spec-python/`](../spec-python/) ni [`../spec-.net/`](../spec-.net/) se modifican: son el
insumo de la evaluación.

Cuando el motor se desvíe del documento de datos, el artículo X manda —«gana el motor»—, y la
corrección del documento se propone como **un cambio explícito y aislado**, nunca como un ajuste
silencioso. Las decisiones que la traducción necesita están en [`consistencia.md`](consistencia.md).

## Orden de lectura

1. [`../spec-python/constitution.md`](../spec-python/constitution.md) — principios innegociables.
2. [`../spec-python/spec.md`](../spec-python/spec.md) — actores, HU, CA, reglas, RNF.
3. Este `README.md`, luego [`plan.md`](plan.md) y [`architecture.md`](architecture.md).
4. [`../spec-python/data-model.md`](../spec-python/data-model.md) y
   [`../spec-python/api-contract.md`](../spec-python/api-contract.md).
5. [`consistencia.md`](consistencia.md) antes de escribir la primera migración.
6. [`tasks.md`](tasks.md) — qué hay que hacer y con qué evidencia se da por hecho.
