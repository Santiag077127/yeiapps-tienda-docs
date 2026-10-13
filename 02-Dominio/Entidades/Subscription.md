# Subscription

## Descripción

`Subscription` representa un período concreto de servicio originado por un [[OrderItem]] confirmado. Conserva fechas y el vínculo con el período anterior cuando es continuación; los recursos usados durante el período viven en [[ResourceAssignmentHistory]]. No equivale a una venta, una cuenta ni una renovación.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del período. |
| `orderItemId` | Ítem comercial que originó el servicio. |
| `startDate` | Primer día del servicio, inclusivo. |
| `expirationDate` | Primer día sin servicio, exclusivo. |
| `previousSubscriptionId` | Período que continúa; opcional, copiado de `OrderItem.previousSubscriptionId`. |
| `status` | Estado calculado por fechas: `ACTIVE` o `EXPIRED`; no se persiste. |
| `resourceAssignmentHistory` | Una asignación inicial y cero o más reemplazos históricos. |

## Duración y ciclo de vida

`serviceDurationDays` se persiste como entero positivo en OrderItem: es una condición comercial contratada y común a las unidades de esa línea. No se duplica en Subscription; su período derivado se conserva en `startDate` y `expirationDate`, y la línea inmutable permite reconstruir el número de días contratado.

Toda Subscription se crea al confirmar completamente su Order. `startDate` es la fecha de negocio de esa confirmación y `expirationDate = startDate + serviceDurationDays`. El intervalo es `[startDate, expirationDate)`: el día inicial cuenta y el día de vencimiento ya no cuenta.

El estado no se guarda ni requiere una transición diaria: para `businessDate`, es `ACTIVE` si `startDate <= businessDate < expirationDate`, y `EXPIRED` si `businessDate >= expirationDate`. Una Subscription futura no se confirma en el MVP. `businessDate` se obtiene usando `store.business_timezone`, parámetro obligatorio del despliegue.

No existe Subscription `PENDING`: `PENDING` es el estado de una Order aún no confirmada; antes de la confirmación no existen Subscription, WalletTransaction de débito ni asignaciones. Una renovación puede prepararse antes de su fecha, pero solo se confirma cuando la fecha de negocio alcanza el vencimiento previo. Una confirmación fallida deja la Order y su línea `PENDING` para reintentar o cancelar.

Al vencer, el estado calculado pasa a `EXPIRED`; no se actualiza la fila, no se libera ResourceAssignmentHistory y no se devuelve automáticamente Account/Profile al stock. La reutilización requiere liberación y publicación administrativa tras el reacondicionamiento externo.

## Relaciones

* Cada Subscription pertenece a exactamente un OrderItem. Al confirmar, cada unidad de `quantity` produce exactamente una Subscription.
* Si el OrderItem tiene `previousSubscriptionId`, Subscription copia el mismo valor. La transacción valida que el período anterior corresponda al mismo Customer y Product, que no tenga ya un sucesor confirmado y que la renovación/continuidad cumpla las reglas del producto.
* Un período admite como máximo un sucesor confirmado por `previousSubscriptionId`. Una referencia solo puede apuntar a una Subscription ya existente; fechas positivas y vínculo inmutable impiden ciclos.
* Toda Subscription llega al COMMIT con exactamente una asignación inicial válida: `INITIAL_ASSIGNMENT`, secuencia 1, recurso compatible con la modalidad. La asignación se crea en la misma transacción.
* Tiene como beneficiario al Seller o Customer de la Order originadora.
* Puede tener una o más filas de ResourceAssignmentHistory: una inicial y cero o más reposiciones.
* La modalidad proviene del snapshot de OrderItem: `PROFILE` usa Profile y `FULL_ACCOUNT` usa Account, incluso en reposiciones.

## Renovación y continuidad

Una renovación para cliente exige que Product la permita; la continuidad con recurso nuevo también queda vinculada al período anterior. En ambos casos se conserva una nueva Order, OrderItem y Subscription, sin cambiar retrospectivamente las fechas del período anterior.

`OrderItem.previousSubscriptionId` conserva el objetivo incluso mientras la Order está `PENDING`. Al confirmar, el mismo vínculo se copia a Subscription. Esta duplicación es intencional: la línea registra la intención comercial reintentable; Subscription registra la sucesión de períodos efectivamente confirmada. Una validación transaccional exige igualdad entre ambos y `uq_subscription_renewal_parent` limita a un sucesor confirmado.

Una línea de continuación tiene `quantity = 1`, porque señala un período anterior concreto. Para continuar varios servicios se usa una línea individual por período anterior. La fecha de confirmación debe ser igual o posterior a `previousSubscription.expirationDate`; `startDate = max(previousSubscription.expirationDate, businessDate)` y, por esa condición, el servicio no se programa a futuro ni se retrodata. Si se conserva el mismo recurso, la transacción libera la asignación anterior y crea la inicial del nuevo período sin consumo adicional de stock. Si se asigna otro recurso, este debe estar disponible; el anterior queda retenido hasta su liberación/publicación manual.

Una garantía no es una renovación: durante el período vigente se agrega una asignación de reemplazo en ResourceAssignmentHistory, sin nueva Order, OrderItem ni Subscription. La garantía conserva el vencimiento original.

## Cancelaciones

El MVP no incluye cancelación de una Subscription confirmada: no existe estado `CANCELLED` ni campos `cancelled_at`, `cancelled_by` o `cancellation_reason`. Solo se cancela una Order antes de confirmar; entonces no existe Subscription que cancelar. Si el negocio requiere terminar anticipadamente un servicio confirmado, deberá definir un CU futuro con política de saldo, efecto de inventario y permisos.

## Ejemplo

Una Subscription anterior vence el 01/10. Una Order de renovación preparada antes de ese día permanece `PENDING`; no cobra ni reserva recurso. Al confirmarla el 01/10 o después, se crea una nueva Subscription que enlaza a la anterior. Si conserva su Profile, la asignación anterior se cierra y la nueva comienza en la misma transacción. Una falla del 13/09 se atiende dentro del período original mediante una reposición, no mediante renovación.

## Fuera del alcance

No procesa pagos externos, no determina precio vigente, no reacondiciona recursos ni automatiza renovación anticipada. La lectura de credenciales se limita a actores autorizados y queda auditada.

## Estado del diseño

**Cerrado para el MVP.** Duración, cálculo de fechas, estados derivados, transición temporal, renovación/continuidad, asignación inicial, vencimiento y alcance de cancelación están definidos. No hay estado futuro de Subscription ni job de mutación al expirar.