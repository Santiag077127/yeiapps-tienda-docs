# Order

## Descripción

`Order` registra una compra comercial y su contexto histórico. Mientras está `PENDING` puede construirse sin [[OrderItem]]; para confirmarse debe contener uno o más. Diferencia explícitamente el vendedor comprador del beneficiario y conserva la información necesaria para reconstruir la operación.

## Responsabilidades

* Agrupar los ítems adquiridos en una compra.
* Conservar buyer, beneficiary, número de pedido, fechas y demás contexto histórico común de la operación.
* Agrupar los snapshots comerciales específicos que pertenecen a sus ítems, sin duplicarlos como datos comunes de la orden.
* Servir de referencia para el débito de billetera o el pago externo, según corresponda.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la orden. |
| `orderNumber` | Número único, aleatorio, permanente y no editable de la orden. |
| `buyerSellerId` | Vendedor que realiza la compra y de cuya billetera sale el dinero. |
| `beneficiary` | Seller comprador o Customer que recibe el servicio. |
| `buyerSnapshot` | Datos históricos estructurados del buyer: identificador, nombre, email y teléfono cuando aplique. |
| `beneficiarySnapshot` | Datos históricos estructurados del beneficiary: identificador, nombre, email y teléfono cuando aplique. |
| `currencyCode` | Moneda ISO 4217 única de toda la orden. |
| `status` | Estado de la orden: `PENDING`, `CONFIRMED` o `CANCELLED`. |
| `createdAt` | Momento de creación. |
| `confirmedAt` | Momento de confirmación, cuando corresponda. |

Los snapshots de buyer y beneficiary se representan con campos estructurados, no como JSONB genérico. Se inicializan al crear la Order con los datos de `User` y se refrescan desde la fuente vigente al confirmar bajo bloqueo. Al confirmar quedan congelados. `User.name`, `email` y `phone` los alimentan; el teléfono puede ser nulo. Cambios posteriores del perfil no reescriben el historial. Producto, oferta, precio, duración, modalidad y cantidad se conservan por [[OrderItem]].

Cada Order tiene una única `currencyCode` ISO 4217, inmutable tras la confirmación. Todos sus OrderItem usan esa misma moneda; en una compra por Wallet coincide también con la moneda de Wallet y de WalletTransaction. Esta igualdad entre registros se valida en la transacción de confirmación, no con un CHECK aislado.

## Estados

`PENDING` representa una operación creada pero todavía no confirmada. `CONFIRMED` consolida las condiciones comerciales; para Seller, el débito de Wallet y la asignación de inventario forman parte de su transacción de confirmación. `CANCELLED` solo aplica antes de la confirmación; no representa un reverso financiero posterior.

Una Order `PENDING` puede existir temporalmente sin ítems mientras se construye, pero no puede pasar a `CONFIRMED` sin al menos un [[OrderItem]]. La confirmación es todo o nada: si falla saldo, precio, inventario o cualquier precondición, la transacción se revierte y la Order permanece `PENDING`, sin débito, Subscription ni asignación parcial.

Transiciones permitidas: `PENDING → CONFIRMED` y `PENDING → CANCELLED`. Seller buyer o Admin puede cancelar solo antes de confirmar y debe aportar razón. AuditEvent conserva actor, hora, Order y razón; no hay columnas de cancelación redundantes. Una Order `CONFIRMED` es histórica e inmutable; no puede volver a `PENDING` ni pasar a `CANCELLED`.

## Relaciones

* Una orden tiene uno o más [[OrderItem]].
* El buyer es un [[Seller]].
* El beneficiary puede ser ese mismo seller o un [[Customer]].
* Puede estar referenciada por un [[WalletTransaction]] o por un [[Payment]], según la modalidad de pago.

## Reglas de negocio relacionadas

* Una orden no se diseña para contener un solo ítem: puede contener múltiples servicios.
* Una orden no tiene un único producto histórico: cada [[OrderItem]] conserva su propio snapshot de producto, oferta, precio, duración, modalidad, cantidad y demás condiciones específicas.
* El dinero siempre sale de la billetera del vendedor que compra.
* La instalación usa una sola moneda configurada; Order, OrderItem y Wallet del buyer deben coincidir.
* Una Order `PENDING` puede cancelarse sin impacto financiero ni de inventario; no existe aún débito, Subscription ni asignación. Una Order `CONFIRMED` no se cancela para representar reembolsos: permanece histórica y toda compensación se registra como movimiento de reverso separado. El MVP no cancela anticipadamente la Subscription confirmada.
* El vendedor solo puede comprar para un customer con relación `SellerCustomer` `ACTIVE` al crear la Order. La relación se valida bajo bloqueo y se referencia junto con el seller comprador; no se crea automáticamente. Inactivarla después no altera la Order histórica.
* La orden confirmada no debe eliminarse físicamente ni modificarse en su contexto comercial.
* Una compra de seller debe coordinar de forma consistente validación de saldo, orden, movimiento, asignación de inventario y suscripción.
* Una orden es una operación comercial; no equivale a la cuenta/perfil ni al período de servicio que se origine.
* Una renovación y una garantía/reposición no deben confundirse: la primera adquiere otro período conforme al [[Product]]; la segunda resuelve un fallo con otro recurso dentro del período vigente, sin nueva orden ni cobro.

## Ejemplos

Si Juan compra para Pedro, `buyer = Seller Juan` y `beneficiary = Customer Pedro`. Si compra para sí mismo, ambos corresponden a Juan como seller. En ambos casos la orden puede incluir varios ítems.

## Fuera del alcance

No mantiene por sí sola el saldo de billetera, no es el inventario asignado y no sustituye la [[Subscription]] que representa el servicio resultante.

## Estado del diseño

**Definido conceptualmente.** Los reversos financieros se registran como [[WalletTransaction]] y no cambian el estado histórico de una Order `CONFIRMED`.
