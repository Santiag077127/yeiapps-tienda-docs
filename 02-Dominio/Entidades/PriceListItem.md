# PriceListItem

## Descripción

`PriceListItem` define el precio vigente de una [[ProductOffer]] dentro de una [[PriceList]].

## Responsabilidades

* Asociar una oferta con una lista de precios.
* Expresar el importe actual que puede aplicarse a una compra.
* Permitir que el [[OrderItem]] guarde una copia del importe usado históricamente.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del ítem. |
| `priceListId` | Lista a la que pertenece. |
| `productOfferId` | Oferta cotizada. |
| `amount` | Importe vigente. |
| `currencyCode` | Moneda ISO 4217 de tres caracteres. |
| `validFrom` | Inicio obligatorio de vigencia. |
| `validTo` | Fin opcional de vigencia, exclusivo. |
| `priority` | Entero positivo para resolver precios coincidentes. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

`amount` usa precisión decimal de hasta cuatro decimales y el redondeo comercial final es a dos decimales. No se usan tipos de punto flotante para dinero.

## Vigencia

PriceListItem no tiene estado independiente: su aplicabilidad se determina por el estado `ACTIVE` de su [[PriceList]], el intervalo de vigencia y su prioridad. El intervalo es semiabierto: `[validFrom, validTo)`. `validFrom` es obligatorio y, cuando existe `validTo`, debe ser posterior a `validFrom`.

## Relaciones

* Cada ítem pertenece a una [[PriceList]].
* Cada ítem cotiza una [[ProductOffer]].
* Un [[OrderItem]] usa el valor aplicable al momento de la compra y lo conserva como histórico.

## Reglas de negocio relacionadas

* No debe colocarse un único precio directamente en [[Product]].
* La modificación de este ítem no puede alterar el precio registrado en órdenes ya realizadas.
* El precio aplicable es el de mayor `priority` entre los ítems vigentes para la lista y la oferta. Un empate de prioridad igualmente aplicable es una configuración ambigua que debe rechazarse o corregirse, nunca resolverse arbitrariamente.
* No pueden solaparse dos ítems de la misma combinación `PriceList`, `ProductOffer` y `priority`; la regla cubre todo solapamiento, no solo intervalos idénticos.
* Los intervalos consecutivos son válidos. Los solapados con prioridades distintas son válidos y prevalece la prioridad mayor, por lo que no queda empate aplicable.
* La implementación PostgreSQL requiere una exclusión temporal mediante `gist` y la extensión `btree_gist` para proteger la combinación anterior.
* Al confirmar una Order, se bloquean y revalidan la lista y los ítems aplicables: oferta, vigencia, prioridad, importe y moneda deben coincidir exactamente con el snapshot de OrderItem.

## Ejemplos

Un ítem de la lista SELLER puede cotizar una oferta de perfil y otro ítem de la lista CUSTOMER puede cotizar la misma oferta con un importe distinto.

## Fuera del alcance

No representa una venta, no determina el beneficiario y no contiene el inventario que se asignará.

## Estado del diseño

**Definido conceptualmente.** Los precios históricos usados por una venta se conservan en [[OrderItem]] aunque la lista cambie posteriormente.
