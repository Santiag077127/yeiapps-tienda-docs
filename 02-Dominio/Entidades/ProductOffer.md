# ProductOffer

## Descripción

`ProductOffer` representa una modalidad comercial específica de un [[Product]]. Es la unidad que se cotiza y se compra, y puede corresponder a `PROFILE` o `FULL_ACCOUNT`.

## Responsabilidades

* Diferenciar las modalidades de venta de un producto.
* Relacionarse con los precios vigentes a través de [[PriceListItem]].
* Expresar la modalidad requerida para asignar inventario.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la oferta. |
| `productId` | Producto al que pertenece. |
| `saleMode` | Modalidad: `PROFILE` o `FULL_ACCOUNT`. |
| `status` | Estado comercial: `ACTIVE` o `INACTIVE`. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

## Estados

| Estado | Significado |
| --- | --- |
| `ACTIVE` | Puede usarse en nuevas compras si Product, Platform, precio y recurso son elegibles. Estado inicial. |
| `INACTIVE` | No puede utilizarse en nuevas compras. |

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. No alteran órdenes, Subscription, inventario ni precios históricos. Una oferta activa no implica stock disponible y su activación no crea inventario.

Toda transición distinta de las anteriores está prohibida. Una oferta solo es comercialmente elegible cuando su Product y Platform están `ACTIVE`, existe una PriceList activa con precio vigente no ambiguo y hay recurso elegible.

## Relaciones

* Cada `ProductOffer` pertenece a un [[Product]].
* Una oferta puede tener precios por lista mediante [[PriceListItem]].
* Un [[OrderItem]] registra la oferta comprada y su modalidad aplicada.

## Reglas de negocio relacionadas

* Las únicas modalidades definidas son `PROFILE` y `FULL_ACCOUNT`.
* Una oferta `FULL_ACCOUNT` solo puede asignar una cuenta sin perfiles previamente asignados.
* Una oferta `PROFILE` asigna un perfil individual disponible.
* `saleMode` es la fuente comercial de verdad. Al crear o confirmar un OrderItem, su `saleModeSnapshot` debe coincidir con ese valor; desde la primera referencia de cualquier OrderItem, incluso PENDING, la modalidad de la oferta no se modifica. Un cambio posterior requiere una ProductOffer nueva.
* No es obligatorio que todo producto tenga ambas modalidades.
* Un mismo Product puede tener ofertas `PROFILE` y `FULL_ACCOUNT` activas simultáneamente; su elegibilidad depende de inventario y no se restringe por compartir Product.
* Para un mismo Product y una misma modalidad puede existir como máximo una ProductOffer `ACTIVE`; las ofertas inactivas se conservan sin bloquear una nueva oferta activa.
* El stock visible de una oferta debe derivarse de inventario disponible; no se define un contador manual como fuente de verdad.
* La duración de un servicio se registra en la operación o servicio correspondiente y no equivale a unidades de inventario.
* La renovabilidad no se determina en la oferta a partir de la cuenta, perfil o precio: pertenece a la configuración explícita del [[Product]].

## Ejemplos

Para un mismo producto, la oferta “Perfil” usa `PROFILE` y la oferta “Cuenta completa” usa `FULL_ACCOUNT`; cada una puede tener precios distintos.

## Fuera del alcance

No es una cuenta ni perfil de inventario, no guarda un precio único y no decide por sí misma la renovabilidad concreta de una suscripción.

## Estado del diseño

**Definido conceptualmente.** Sus estados, modalidades y unicidad comercial están cerrados; la renovabilidad se configura en [[Product]].
