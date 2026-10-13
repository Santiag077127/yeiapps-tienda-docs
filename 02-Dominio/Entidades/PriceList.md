# PriceList

## Descripción

`PriceList` agrupa precios vigentes para un tipo de comprador. El sistema debe soportar, como mínimo, una lista para `SELLER` y otra para `CUSTOMER`.

## Responsabilidades

* Identificar el contexto de precio aplicable.
* Agrupar los [[PriceListItem]] que definen precios actuales de ofertas.
* Separar el precio de catálogo del precio histórico de una orden.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la lista. |
| `name` | Nombre de la lista. |
| `targetRole` | Rol al que aplica: `SELLER` o `CUSTOMER`. |
| `status` | Estado operativo: `ACTIVE` o `INACTIVE`. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

## Estados

| Estado | Significado |
| --- | --- |
| `ACTIVE` | Puede participar en el cálculo de precio si corresponde al rol del comprador y tiene un ítem vigente no ambiguo. Estado inicial. |
| `INACTIVE` | No participa en nuevas determinaciones de precio. |

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. La vigencia comercial se define en cada [[PriceListItem]] mediante `validFrom`, `validTo` y `priority`; activar una lista no vuelve válidos automáticamente todos sus ítems.

Toda transición distinta de las anteriores está prohibida. Una lista `ACTIVE` solo participa si su `targetRole` corresponde al comprador y existe un PriceListItem vigente con selección no ambigua.

## Relaciones

* Una `PriceList` contiene cero o más [[PriceListItem]].
* Cada ítem cotiza una [[ProductOffer]].
* Un [[OrderItem]] conserva el valor aplicado, independiente de cambios posteriores en la lista.

## Reglas de negocio relacionadas

* Deben existir como mínimo contextos de precio para `SELLER` y `CUSTOMER`.
* Para cada `targetRole` puede existir como máximo una lista `ACTIVE`. Su sustitución se confirma en una sola transacción que bloquea las listas del rol y revalida la unicidad; no puede quedar un estado confirmado con dos listas activas.
* El precio aplicado depende del tipo de comprador y de una lista `ACTIVE` aplicable.
* Cambiar precios vigentes no modifica precios históricos de órdenes.
* La lista aplicable se bloquea y revalida al confirmar la Order; el instante de determinación comercial del precio es la confirmación.

## Ejemplos

La lista de SELLER y la lista de CUSTOMER pueden contener importes distintos para la misma oferta de perfil.

## Fuera del alcance

No asigna inventario, no procesa pagos y no reemplaza el precio histórico registrado en [[OrderItem]].

## Estado del diseño

**Definido conceptualmente.** Inactivar o reactivar una lista no altera Orders, precios históricos ni Subscription vigentes.
