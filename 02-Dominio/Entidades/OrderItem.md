# OrderItem

## Descripción

`OrderItem` representa un servicio/oferta individual dentro de una [[Order]]. Conserva el snapshot comercial y, cuando la línea es una renovación o continuidad, identifica el período anterior mientras la orden está pendiente. El recurso operativo usado para prestar el servicio pertenece a [[Subscription]] y su [[ResourceAssignmentHistory]].

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del ítem. |
| `orderId` | Orden a la que pertenece. |
| `productId` | Referencia al producto utilizado en la venta. |
| `productNameSnapshot` | Nombre histórico del producto vendido. |
| `productOfferId` | Oferta utilizada, perteneciente al mismo Product. |
| `unitPrice` | Precio unitario histórico. |
| `currencyCode` | Moneda ISO 4217 de la línea. |
| `saleModeSnapshot` | Modalidad histórica: `PROFILE` o `FULL_ACCOUNT`. |
| `quantity` | Cantidad de unidades adquiridas; entero positivo. |
| `serviceDurationDays` | Duración comercial histórica en días completos; entero positivo. |
| `previousSubscriptionId` | Período anterior que la línea renueva/continúa; opcional. |

Los datos comerciales se conservan como campos estructurados, no como JSONB genérico. `unitPrice` usa `NUMERIC(19,4)` y el redondeo comercial final es a dos decimales. `lineTotal` se deriva de `quantity × unitPrice`; no se persiste.

## Cantidad, confirmación y servicio

`quantity` representa unidades independientes con iguales oferta, precio, moneda, modalidad y duración. Mientras la Order está `PENDING`, no existe ninguna Subscription ni asignación originada por el ítem. Al confirmar la Order, cada unidad crea exactamente una Subscription y una asignación inicial válida en la misma transacción.

La confirmación es todo o nada para todos los ítems: si saldo, precio, credenciales, compatibilidad o inventario no alcanza para una unidad, se revierte el débito, todas las Subscriptions y asignaciones; la Order permanece `PENDING` sin efecto parcial. `serviceDurationDays` permanece en OrderItem porque es el término comercial vendido y una duración común a la línea; Subscription conserva sus límites de vigencia calculados, no una copia de la duración.

Si `previousSubscriptionId` está presente, `quantity` debe ser 1: la línea se refiere a un solo servicio anterior. Varios servicios continuados usan líneas separadas. La referencia se conserva en borrador y, al confirmar, se copia a la Subscription resultante. La transacción verifica beneficiario, Product, fecha efectiva y que el período anterior no tenga sucesor confirmado.

## Reglas de negocio

* Precio, moneda, producto, oferta, duración, modalidad, cantidad y referencia al período anterior no cambian después de confirmar la Order.
* `saleModeSnapshot` coincide con `ProductOffer.saleMode` al crear y confirmar la Order.
* `PROFILE` asigna un Profile disponible; `FULL_ACCOUNT` asigna una Account elegible y bloquea sus Profiles.
* Una línea sin `previousSubscriptionId` es una adquisición regular; una línea con ese campo es una renovación/continuidad y requiere una fecha de confirmación que no preceda el vencimiento del período anterior.
* Un `previousSubscriptionId` en una Order `PENDING` no reserva recurso ni genera cargo. Si se cancela la Order, no se crea Subscription y el vínculo permanece como parte del intento comercial histórico.
* El recurso puede cambiar por garantía sin alterar el snapshot comercial del ítem.

## Relaciones

* Cada ítem pertenece a una Order y referencia un Product y una ProductOffer coherentes.
* Un ítem confirmado origina exactamente `quantity` Subscriptions; un ítem pendiente o de una Order cancelada origina cero.
* Cada Subscription conserva el `previousSubscriptionId` del OrderItem, si existe.

## Fuera del alcance

No mantiene el saldo, no representa la asignación operativa ni define por sí solo una política de devolución. Impuestos, descuentos y promociones no forman parte del MVP.

## Estado del diseño

**Cerrado para el MVP.** Snapshot, cantidad, vínculo de continuidad, relación con Subscription y atomicidad están definidos.