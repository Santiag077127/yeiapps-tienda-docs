# Product

## Descripción

`Product` representa el producto comercial del catálogo. Se diferencia del inventario: una [[Account]] o un [[Profile]] es inventario; el producto define qué se ofrece comercialmente.

## Responsabilidades

* Identificar un concepto comercializable.
* Agrupar las [[ProductOffer]] comercialmente publicadas.
* Mantener la referencia conceptual a la plataforma o servicio que comercializa.
* Configurar explícitamente las reglas de renovabilidad que aplican a los servicios comercializados.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del producto. |
| `name` | Nombre comercial. |
| `platformId` | Plataforma obligatoria a la que pertenece el producto. |
| `customerRenewable` | Indica si el cliente puede adquirir otro período del servicio. |
| `internallyRenewable` | Indica si el negocio puede reacondicionar/reutilizar manualmente el recurso para una nueva comercialización. |
| `status` | Estado comercial: `ACTIVE` o `INACTIVE`. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

## Estados

| Estado | Significado |
| --- | --- |
| `ACTIVE` | Puede comercializarse si Platform, ProductOffer, PriceListItem y recurso son elegibles. Estado inicial. |
| `INACTIVE` | No permite nuevas compras ni asignaciones comerciales. |

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. La inactivación no cambia ofertas, listas, órdenes, Subscription ni asignaciones históricas; la reactivación no altera snapshots ni precios históricos.

Toda transición distinta de las anteriores está prohibida. Un Product `ACTIVE` solo es comercialmente elegible si su Platform también está `ACTIVE`, existe una oferta activa, un precio válido aplicable y recurso elegible.

## Relaciones

* Un `Product` puede tener cero o más [[ProductOffer]].
* Pertenece a exactamente una [[Platform]].
* Un [[OrderItem]] conserva la referencia histórica al producto adquirido.

## Reglas de negocio relacionadas

* Un producto no debe asumir un único precio.
* No todos los productos tienen necesariamente ambas modalidades de venta.
* El precio actual se configura mediante [[PriceList]] y [[PriceListItem]].
* El producto no representa stock ni un recurso de inventario; la disponibilidad comercial se deriva de cuentas y perfiles elegibles.
* La renovabilidad se configura explícitamente en el producto, no se deduce de una [[Account]], [[Profile]] ni de su historial.
* `customerRenewable` e `internallyRenewable` son configuraciones independientes: una no implica la otra.
* La renovabilidad es independiente del precio y no sustituye [[PriceList]] ni [[PriceListItem]].
* La renovabilidad interna no automatiza acciones sobre cuentas o perfiles: el reacondicionamiento operativo ocurre fuera de la plataforma y el recurso solo vuelve al inventario cuando se carga/publica nuevamente.
* Todo recurso asignado a una venta debe pertenecer a la misma Platform del Product de su [[ProductOffer]]: para `PROFILE`, la Platform de la Account del Profile; para `FULL_ACCOUNT`, la Platform de la Account asignada.

## Ejemplos

Un producto comercial “Netflix” puede tener una oferta de perfil y otra de cuenta completa; ambas son [[ProductOffer]] separadas.

| Ejemplo de producto | Renovable para cliente | Renovable internamente |
| --- | ---: | ---: |
| Streaming por perfil | No | Sí |
| Cuenta completa | Sí | Sí |
| Producto no reutilizable | No | No |

La tabla ilustra configuraciones posibles; no es un catálogo definitivo.

## Fuera del alcance

No representa una cuenta o perfil concreto, ni almacena el precio único de una venta ni el inventario asignado.

## Estado del diseño

**Definido conceptualmente.** La Platform es obligatoria y la compatibilidad Product/Offer/recurso se valida durante la asignación.
