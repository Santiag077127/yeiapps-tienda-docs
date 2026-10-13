# Reglas de Negocio

Este documento contiene las reglas funcionales que deben cumplirse
independientemente de la interfaz utilizada por el usuario.

---

# 1. Usuarios y roles

## RN-001 — Roles del sistema

El sistema tendrá tres roles principales:

- ADMIN
- SELLER
- CUSTOMER

---

## RN-002 — Separación de permisos

Cada usuario solamente podrá ejecutar las operaciones autorizadas
para su rol.

Las restricciones deben validarse en el backend y no únicamente
en el frontend.

---

# 2. Seller

## RN-003 — Wallet del Seller

Cada Seller dispone de una Wallet para realizar compras dentro
de la plataforma.

---

## RN-004 — Recarga de Wallet

La Wallet de un Seller puede ser recargada por un Admin.

Toda recarga debe generar un registro histórico de la operación.

---

## RN-005 — Saldo no negativo

Una Wallet no puede quedar con saldo negativo.

Una compra solamente puede realizarse si existe saldo suficiente.

---

## RN-006 — Historial financiero

Los movimientos de Wallet no deben eliminarse físicamente.

Cada movimiento debe conservar:

- Tipo de movimiento
- Valor
- Saldo anterior
- Saldo posterior
- Fecha
- Referencia de la operación
- Usuario que originó la operación

---

# 3. Inventario

## RN-007 — Platform

Una Platform representa un servicio de streaming comercializado
por la plataforma.

Ejemplos:

- Netflix
- Disney+
- Spotify

Los ejemplos anteriores son únicamente ilustrativos.

---

## RN-008 — Account

Una Account representa una cuenta perteneciente a una Platform.

Una Account puede contener uno o varios Profiles.

Sus estados conceptuales son `AVAILABLE`, `PARTIALLY_ASSIGNED`, `FULLY_ASSIGNED` e `INACTIVE`. `EXPIRED` no aplica a Account; el vencimiento pertenece a Subscription.

---

## RN-009 — Profile

Un Profile pertenece a una Account.

Un Profile disponible puede ser asignado mediante una venta.

Su disponibilidad derivada es `AVAILABLE`, `ASSIGNED`, `BLOCKED_BY_FULL_ACCOUNT` o `INACTIVE`. Solo `isActive` se persiste; `EXPIRED` no aplica a Profile, pues el vencimiento pertenece a Subscription.

---

## RN-010 — Profile no reasignable durante su servicio

Un Profile asignado no puede volver a venderse ni asignarse a otro cliente mientras conserve esa asignación de servicio.

El vencimiento del servicio no devuelve automáticamente el Profile al stock ni hace que la plataforma lo reacondicione. El reacondicionamiento lo realiza el negocio manualmente fuera de la plataforma; solo después el administrador puede cargar/preparar nuevamente el recurso en inventario para que vuelva a estar disponible.

---

# 4. Venta de Account completa

## RN-011 — Modalidades de venta

Una Account puede comercializarse mediante dos modalidades:

- PROFILE
- FULL_ACCOUNT

---

## RN-012 — Venta de Profile

Cuando se vende un Profile:

- El Profile queda asignado.
- El Profile deja de estar disponible.
- Se registra la venta.
- Se registra la suscripción correspondiente.

Los demás Profiles de la Account pueden continuar disponibles.

---

## RN-013 — Venta de Account completa

Una Account completa solamente puede venderse si ninguno de sus
Profiles ha sido asignado previamente.

---

## RN-014 — Bloqueo de Profiles

Cuando una Account completa es vendida:

- La Account pasa a `FULLY_ASSIGNED`.
- Todos sus Profiles dejan de estar disponibles para nuevas ventas/asignaciones por la condición derivada `Account = FULLY_ASSIGNED`.
- No se pueden realizar nuevas ventas de sus Profiles.

Esta condición no crea una columna ni un estado persistido. La vista sí presenta `BLOCKED_BY_FULL_ACCOUNT` para explicar la indisponibilidad; las otras disponibilidades derivadas son `AVAILABLE`, `ASSIGNED` e `INACTIVE`.

---

## RN-015 — Incompatibilidad de modalidades

No se puede vender una Account completa si alguno de sus Profiles
ya fue vendido.

No se puede vender un Profile de una Account que ya fue vendida
completamente.

---

# 5. Productos y precios

## RN-016 — Producto

Un Product representa aquello que se comercializa dentro del Store.

---

## RN-017 — Ofertas

Un Product puede tener diferentes modalidades de comercialización.

Por ejemplo:

- Venta de Profile
- Venta de Account completa

---

## RN-018 — Listas de precios

El sistema debe permitir diferentes listas de precios.

Como mínimo existirán:

- Seller Price
- Customer Price

---

## RN-019 — Precio según actor

El precio aplicado depende del tipo de comprador.

Un Seller utiliza el precio correspondiente a Seller.

Un Customer utiliza el precio correspondiente a Customer.

---

## RN-020 — Precio histórico

El precio utilizado en una venta debe quedar registrado en la Order.

Los cambios posteriores de precio no deben modificar el valor
histórico de una venta.

---

# 6. Orders

## RN-021 — Registro de venta

Toda compra confirmada debe generar una Order.

---

## RN-022 — Order histórica

Una Order confirmada no debe eliminarse físicamente. Seller buyer o Admin puede cancelar una Order PENDING, con razón obligatoria registrada en AuditEvent. Esta acción no debita ni asigna recursos. Una Order confirmada no se cancela; cualquier corrección financiera se anota como WalletTransaction.

---

## RN-023 — Estado de Order

La definición normativa y única de estados y transiciones de Order está en **RN-068**. Para el MVP son exclusivamente `PENDING`, `CONFIRMED` y `CANCELLED`, con transiciones `PENDING → CONFIRMED` y `PENDING → CANCELLED`. Un reverso, ajuste o devolución financiera se registra mediante `WalletTransaction`; no añade ni cambia un estado histórico de Order.

---

# 7. Wallet y Order

## RN-024 — Compra Seller

Una compra realizada por un Seller debe:

1. Validar el Seller.
2. Obtener el precio correspondiente.
3. Validar el saldo disponible.
4. Crear/confirmar la Order.
5. Registrar el movimiento de Wallet.
6. Asignar el inventario.
7. Crear la suscripción correspondiente.

Estas operaciones deben ejecutarse de forma consistente.

---

# 8. Suscripciones

## RN-025 — Subscription

Una Order confirmada crea exactamente una Subscription por unidad contratada cuando el Product representa servicio con duración. No se crea Subscription para una Order pendiente o cancelada.

---

## RN-026 — Fecha y duración del servicio

La duración se conserva como entero positivo de días en OrderItem. Al confirmar, `startDate` es la fecha de negocio actual y `expirationDate = startDate + serviceDurationDays`. `startDate` es inclusiva y `expirationDate` exclusiva.

---

## RN-027 — Servicio expirado

El estado de servicio es derivado, no persistido: `ACTIVE` para `startDate <= businessDate < expirationDate`; `EXPIRED` para `businessDate >= expirationDate`. No se necesita job que actualice Subscription. Expirar no elimina el servicio histórico, no libera Account/Profile ni devuelve stock automáticamente.

---
## RN-028 — Customer sin Wallet

El Customer no dispone de Wallet.

---

## RN-029 — Pago del Customer

Las compras realizadas por Customer se procesarán mediante
un sistema de pagos externo.

La integración concreta con el proveedor será definida
posteriormente.

---

# 10. Historial y eliminación

## RN-030 — Protección del historial

Las entidades relacionadas con operaciones financieras,
ventas y suscripciones no deben eliminarse físicamente cuando
su eliminación implique perder información histórica.

---

## RN-031 — Estados en lugar de eliminación

Cuando sea necesario retirar una entidad del funcionamiento
activo, se debe evaluar utilizar:

- Estado
- Desactivación
- Soft delete
- Cancelación

en lugar de eliminar físicamente el registro.

---

# 11. Credenciales

## RN-032 — Protección de credenciales

Las credenciales de las cuentas de streaming son información
sensible.

No deben exponerse indiscriminadamente mediante las respuestas
normales de la API.

El acceso a credenciales deberá estar controlado y, cuando
corresponda, auditado.

---

# 12. Auditoría

## RN-033 — Operaciones importantes

Las operaciones críticas deberán permitir identificar:

- Quién realizó la operación.
- Cuándo se realizó.
- Qué operación se ejecutó.
- Qué entidad fue afectada.

La auditoría mínima usa `AuditEvent`: actor USER o SYSTEM, acción, objeto, fecha UTC, cambios no sensibles, razón/contexto seguro y request ID opcional. Audita altas/bajas/cambios de usuarios y roles, catálogo/precios, credenciales, cancelación de Order, renovaciones, reposiciones, asignaciones/liberaciones y operaciones administrativas; también cada lectura de credenciales. WalletTransaction conserva como registro canónico actor, fecha, importe, saldos y secuencia de cada recarga, débito, ajuste y reverso. ResourceAssignmentHistory conserva actor, fechas y motivos de asignación/liberación. No se registran contraseñas, hashes, tokens ni secretos en AuditEvent; el log es append-only para el rol de aplicación.

---

# 13. Stock, asignaciones y control de cuentas

## RN-034 — Stock para nuevas asignaciones

El stock representa recursos de inventario disponibles para nuevas asignaciones. Debe derivarse de la disponibilidad real de Accounts y Profiles elegibles para la modalidad comercial; no es un número manual independiente de inventario.

---

## RN-035 — Consumo de disponibilidad

Una nueva asignación de Account o Profile consume disponibilidad para nuevas asignaciones. La duración del servicio no altera la cantidad de recursos consumidos.

---

## RN-036 — Account, Order y Subscription son distintos

Una Account es un recurso de inventario, una Order es una operación comercial y una Subscription representa un período de servicio. Una cuenta puede participar en varios períodos de servicio a lo largo del tiempo.

---

## RN-037 — Inventario histórico

Los recursos asignados no se eliminan del inventario histórico. Una Account vendida completamente permanece disponible para consulta, aunque deje de estar disponible para nuevas ventas.

---

## RN-038 — Stock por modalidad

Una venta FULL_ACCOUNT consume una cuenta como un único recurso; sus perfiles no se cuentan como cuentas independientes. Para PROFILE, stock es el número de Profiles `AVAILABLE` en `profile_availability`; para FULL_ACCOUNT, stock es el número de Accounts `AVAILABLE` en `account_availability`. Los contadores se presentan separados por modalidad.

Para `FULL_ACCOUNT`, solo las Accounts en `AVAILABLE` son elegibles. Para `PROFILE`, el stock corresponde a Profiles en `AVAILABLE`; una Account `PARTIALLY_ASSIGNED` puede seguir aportando perfiles disponibles.

---

# 14. Renovaciones y duración

## RN-039 — Renovación con el mismo recurso

Una renovación de cliente usa el mismo Account o Profile ya asignado únicamente cuando el Product está configurado como renovable para cliente y las reglas operativas permitan conservar el recurso.

---

## RN-040 — Renovación sin nuevo consumo de stock

La renovación genera un nuevo período mediante una nueva Subscription y no modifica la Subscription anterior. Si conserva el mismo recurso, no consume una nueva unidad de stock.

---

## RN-041 — Continuidad sin renovación para cliente

Si el Product no está configurado como renovable para cliente y el beneficiario desea continuar el servicio, se requiere una nueva asignación de recurso disponible. Esa nueva asignación sí consume disponibilidad.

---

## RN-042 — Duración variable

La duración del servicio puede variar entre compras o períodos. Debe conservarse en la operación o servicio correspondiente y no se debe asumir una duración fija.

---

# 15. Control de cuentas

## RN-043 — Consulta operativa de recursos asignados

El sistema debe permitir un Control de Cuentas que consulte recursos asignados y sus servicios, incluyendo número de pedido, seller, customer, nombre, correo y teléfono cuando aplique, producto, cuenta/plataforma/perfil cuando aplique, datos operativos de acceso según rol y permisos, fechas de inicio y vencimiento, duración, días restantes, estado, modalidad y datos de renovación o garantía disponibles.

---

## RN-044 — Estado independiente del recurso y del servicio

El estado del servicio —por ejemplo, ACTIVE o EXPIRED— es independiente del historial y existencia del recurso de inventario. La expiración no elimina la Account ni el Profile.

---

## RN-045 — Reutilización posterior a expiración

La reutilización interna de una Account o Profile depende de que el Product esté configurado como renovable internamente. La plataforma no devuelve automáticamente el recurso al stock: el reacondicionamiento ocurre manualmente fuera de ella y el administrador debe cargar/publicar nuevamente el recurso para que vuelva a estar disponible.

---

# 16. Configuración de renovabilidad y garantía

## RN-046 — Renovabilidad configurable por Product

El administrador configura explícitamente en Product si el servicio es renovable para cliente y si es renovable internamente. Estas reglas no se determinan automáticamente analizando Account, Profile o historial.

---

## RN-047 — Independencia de renovabilidades

La renovabilidad para cliente y la renovabilidad interna son conceptos independientes. Que el negocio pueda reacondicionar/reutilizar un recurso no habilita por sí mismo que el cliente adquiera otro período del mismo servicio.

---

## RN-048 — Renovación de cliente

La renovación de cliente es la adquisición de otro período de servicio conforme a la configuración del Product. Si conserva el mismo recurso, no crea una cuenta nueva ni consume nuevamente stock.

---

## RN-049 — Reutilización interna

Cuando un Product es renovable internamente, el negocio puede reacondicionar manualmente el recurso una vez terminado el período para una futura comercialización. La plataforma no cambia credenciales, sesiones, perfiles ni disponibilidad automáticamente; el recurso no se elimina y solo vuelve al inventario tras una nueva carga/publicación administrativa.

---

## RN-050 — Garantía o reposición

La entrega de otro recurso para resolver una falla durante el período contratado es una garantía o reposición, no una renovación. Su registro se realiza mediante ResourceAssignmentHistory dentro de la Subscription existente.

---

# 17. Garantías y reposiciones

## RN-051 — Garantía durante período vigente

Una garantía ocurre cuando el recurso entregado falla durante un período de servicio vigente. El negocio sustituye ese recurso para solucionar la falla.

---

## RN-052 — Conservación del período original

Una garantía no inicia un período comercial nuevo, no reinicia los días y no extiende el vencimiento. El recurso de reemplazo conserva la fecha de finalización y los días restantes del servicio original.

---

## RN-053 — Sin nueva compra ni cobro

Una garantía no genera una nueva Order ni una nueva compra del cliente, y no se cobra nuevamente por el recurso de reemplazo.

---

## RN-054 — Reemplazos admisibles

Para un servicio de cuenta completa, una cuenta elegible puede reemplazar la cuenta fallida. Para un servicio de perfil, el reemplazo puede ser otro perfil de la misma cuenta o un perfil elegible de otra cuenta.

---

## RN-055 — Recurso defectuoso y disponibilidad

Al producirse una garantía, el recurso defectuoso queda no disponible para nuevas ventas y pasa a revisión/diagnóstico. No retorna automáticamente al stock; su reparación, reacondicionamiento o destino posterior están pendientes.

---

## RN-056 — Reasignación dentro del servicio

La garantía agrega un reemplazo al historial de asignaciones de la Subscription existente y conserva su período. Debe poder rastrearse el recurso original, la falla, el reemplazo, beneficiario, vendedor de origen, días restantes y momento de reposición. No existe un límite definido de garantías mientras el servicio esté vigente y la falla corresponda a garantía.

---

## RN-057 — Disponibilidad en la garantía

El stock refleja inmediatamente la asignación vigente del reemplazo y la liberación del recurso fallido. Account/Profile defectuosos se marcan administrativamente inactivos mientras se revisan; no se reincorporan hasta reacondicionamiento y publicación manual.

---

# 18. Operación manual e historial comercial

## RN-058 — Vencimiento sin retorno automático

El vencimiento de una Subscription solo registra que el servicio deja de estar vigente. No devuelve automáticamente la Account o Profile al stock ni la marca como lista para una nueva venta.

---

## RN-059 — Reacondicionamiento externo y recarga

El reacondicionamiento físico/operativo de cuentas y perfiles se realiza manualmente fuera de la plataforma. Solo después de ese proceso el administrador puede volver a cargar/publicar un recurso para que integre el inventario disponible.

---

## RN-060 — Pedido histórico inmutable

El número de pedido es único, aleatorio, permanente y no editable. Los datos históricos principales de la operación —pedido, seller, customer/beneficiario, nombre, correo, teléfono cuando aplique, producto y fecha del servicio— son inmutables.

---

## RN-061 — Datos operativos modificables

Los datos operativos del recurso, como correo de acceso, contraseña, perfil y estado operativo, pueden cambiar cuando corresponda. Esos cambios no modifican los datos históricos/comerciales de la Order.

---

# 19. Estados de inventario

## RN-062 — Estados de Account

Account persiste solo `isActive`. La vista deriva `INACTIVE` si la habilitación es falsa o faltan credenciales comerciales; `AVAILABLE` si hay capacidad y no existe asignación completa; `PARTIALLY_ASSIGNED` si quedan Profiles elegibles y otros están asignados; `FULLY_ASSIGNED` cuando no queda capacidad elegible por asignaciones vigentes. `EXPIRED` no es un estado de Account.

---

## RN-063 — Estados de Profile

Profile persiste solo `isActive`. La vista deriva `INACTIVE`, `ASSIGNED`, `BLOCKED_BY_FULL_ACCOUNT` o `AVAILABLE` desde habilitación, credenciales de Account, cuenta padre y asignación vigente. `EXPIRED` no es un estado válido de Profile y un vencimiento no devuelve automáticamente el recurso al inventario.

---

# 20. Períodos, renovaciones y asignaciones históricas

## RN-064 — Subscription como período concreto

Cada Subscription representa un período originado por OrderItem. `serviceDurationDays` es entero positivo persistido en la línea comercial; `startDate` y `expirationDate` se persisten en Subscription y delimitan `[startDate, expirationDate)`. El período nace al confirmar la Order y comienza en `businessDate`. Su estado `ACTIVE`/`EXPIRED` se calcula desde esas fechas y la zona horaria configurada, no se persiste.

---

## RN-065 — Renovación/continuidad como nueva operación

La continuidad crea nueva Order, OrderItem y Subscription sin modificar el período anterior. OrderItem conserva `previousSubscriptionId` desde el borrador; Subscription copia el enlace al confirmar. Beneficiario y Product deben coincidir. Cada período anterior admite como máximo un sucesor confirmado; la inserción solo puede referir una Subscription preexistente y el vínculo es inmutable.

Una línea con `previousSubscriptionId` tiene `quantity = 1`. Se puede preparar antes del vencimiento, pero solo confirmar cuando `businessDate >= previousSubscription.expirationDate`; `startDate = max(expirationDate anterior, businessDate)`. No se cobra ni reserva recurso antes de confirmar, no hay período futuro ni retroactivo. Si se conserva recurso, la asignación anterior se libera con motivo `RENEWAL_TRANSFER` y la nueva se crea atómicamente; si se asigna recurso distinto, el anterior permanece retenido hasta la republicación manual.

---

## RN-066 — Historial de asignaciones de recurso

Cada Subscription persistida al COMMIT tiene exactamente una `INITIAL_ASSIGNMENT` de secuencia 1 y cero o más `WARRANTY_REPLACEMENT`. La asignación inicial se crea en la misma transacción. Cada fila identifica recurso, momento, secuencia y actor. La reposición conserva en la fila nueva actor y datos mínimos de falla; la fila previa identifica el recurso fallido. La liberación guarda timestamp, actor y motivo (`WARRANTY_REPLACEMENT`, `RENEWAL_TRANSFER` o `MANUAL_RECONDITIONING`) y se completa una sola vez.

---

## RN-067 — Snapshot comercial por ítem

Order conserva solo el snapshot común. OrderItem conserva Product, oferta, precio, moneda, duración, modalidad, cantidad y, si existe, Subscription previa. Los recursos usados son datos operativos de Subscription y ResourceAssignmentHistory, no parte del snapshot comercial.

---

## RN-068 — Estados y cancelaciones de Order/Subscription

Order tiene estados `PENDING`, `CONFIRMED` y `CANCELLED`, con transiciones `PENDING → CONFIRMED` y `PENDING → CANCELLED`. Seller buyer o Admin puede cancelar una Order PENDING, con razón obligatoria. Actor, fecha y razón se registran en AuditEvent; no se duplican en columnas de cancelación. Una Order CONFIRMED es histórica y no se cancela.

Subscription no persiste estado: `ACTIVE` y `EXPIRED` son derivados de fechas según RN-027. No tiene estado `PENDING` ni `CANCELLED`; una cancelación anticipada de servicio confirmado queda fuera del MVP. El vencimiento no cancela la Order, no crea reembolso y no libera recurso. Correcciones financieras se anotan como movimiento Wallet separado.

---
## RN-069 — Confirmación financiera y concurrencia

La compra se confirma en una transacción PostgreSQL `READ COMMITTED` con locking pesimista y revalidación: se bloquean recurso candidato y Wallet, se valida disponibilidad, Platform y saldo, y se persisten Order, Subscription, ResourceAssignmentHistory, estados y WalletTransaction. Cualquier fallo produce rollback completo. No existe Reservation en el MVP.

Las escrituras de Account, Profile y ResourceAssignmentHistory no se realizan directamente por la aplicación. Se ejecutan mediante funciones PostgreSQL autorizadas que bloquean las Accounts raíz antes de modificar filas hijas, según el protocolo físico; esto evita inversiones de locks causadas por los locks automáticos de `UPDATE` y `DELETE`.

---

## RN-070 — Compatibilidad de plataforma

Todo Product pertenece a una Platform. El recurso usado por una venta debe pertenecer a esa misma Platform: Account directamente para `FULL_ACCOUNT` y Account propietaria del Profile para `PROFILE`.

---

## RN-071 — Estados de identidad, catálogo y relaciones

Platform, Product, ProductOffer, PriceList, User, Seller, Customer y SellerCustomer usan exclusivamente `ACTIVE` e `INACTIVE`. Cada entidad nace `ACTIVE` y permite `ACTIVE → INACTIVE` e `INACTIVE → ACTIVE`. La inactivación bloquea nuevas operaciones que dependan de la entidad, pero no elimina historial, no cancela Subscription vigentes ni modifica automáticamente estados relacionados.

---

## RN-072 — Elegibilidad comercial compuesta

Una nueva compra de Seller para Customer exige User de Seller activo, Seller activo, User de Customer activo, Customer activo, SellerCustomer activo, Platform, Product, ProductOffer y PriceList activos, precio vigente no ambiguo, inventario elegible y Wallet con saldo suficiente y moneda compatible. Ningún estado individual reemplaza esas validaciones.

---

## RN-073 — Inactivación sin cascadas

Inactivar Platform no inactiva automáticamente Product, Account ni Profile; inactivar Product no inactiva ofertas ni listas; inactivar Seller no modifica Wallet; e inactivar SellerCustomer no cancela Subscription. En todos los casos la elegibilidad futura se bloquea al evaluar las entidades relacionadas.

---

## RN-074 — Publicación de precios sin ambigüedad

Por cada rol objetivo existe como máximo una PriceList activa. PriceListItem usa vigencia semiabierta `[validFrom, validTo)`, inicio obligatorio, final opcional posterior e `priority` positiva. Para la misma lista, oferta y prioridad no se permiten intervalos solapados; los consecutivos sí. Si coinciden precios de prioridades diferentes, prevalece la mayor.

---

## RN-075 — Moneda y referencia de compra

Cada Order tiene una única moneda ISO y todos sus OrderItem usan esa misma moneda. En una compra debitada desde Wallet, Wallet, Order y WalletTransaction usan la misma moneda. `PURCHASE_DEBIT` referencia obligatoriamente la Order; ningún otro tipo de WalletTransaction la referencia directamente. Estas igualdades se validan bajo la transacción bloqueada de confirmación.

---

## RN-076 — Reverso financiero total y trazable

Un WalletTransaction original admite a lo sumo un reverso total. Solo TOP_UP, PURCHASE_DEBIT y los dos ADJUSTMENT pueden revertirse; REVERSAL no. El nuevo REVERSAL pertenece a la misma Wallet, usa igual moneda y tiene el importe opuesto; no modifica el original. Para una corrección posterior se crea un ajuste administrativo autorizado.

---

## RN-077 — Integridad de Product, beneficiario e inventario

Un OrderItem no puede combinar un Product con una ProductOffer de otro Product. Para crear una Order con Customer beneficiario, SellerCustomer debe existir y estar ACTIVE bajo bloqueo; no se crea de manera implícita. Una Account debe crearse con al menos un Profile y requiere credenciales de streaming completas y válidas para ser comercialmente elegible, sin introducir estados adicionales.

---

## RN-078 — Historial e inmutabilidad física

WalletTransaction no se actualiza ni elimina. Una Order CONFIRMED, sus snapshots y sus OrderItem no se alteran. ResourceAssignmentHistory es append-only excepto por completar `releasedAt` de la fila vigente. Permisos mínimos y triggers de base de datos refuerzan estas reglas, así como las transiciones permitidas de Order y Subscription y el cambio de saldo solo junto al nuevo movimiento financiero.

---

## RN-079 — Motivos cerrados de asignación

ResourceAssignmentHistory solo admite `INITIAL_ASSIGNMENT` y `WARRANTY_REPLACEMENT`. La primera corresponde al inicio del servicio; el reemplazo requiere una asignación previa liberada y conserva el vencimiento de la misma Subscription. La validación de secuencia y contexto se realiza en la transacción de asignación.

---

## RN-080 — Wallet del buyer y débito idempotente

Una compra de Seller solo puede financiarse con la Wallet cuyo titular es el Seller buyer de la Order. Una Order admite como máximo un `PURCHASE_DEBIT`; reintentos concurrentes no pueden generar un segundo descuento. La confirmación bloquea Order y Wallet, revalida propietario, moneda y saldo, y confirma o revierte toda la operación.

---

## RN-081 — Saldo materializado y ledger

El saldo de Wallet es el último saldo resultante de su historial, o cero cuando no existen movimientos. Cada WalletTransaction conserva saldo inmediatamente anterior, delta firmado y saldo inmediatamente posterior. Los movimientos de una Wallet se serializan mediante bloqueo pesimista; no existe actualización de saldo sin el movimiento financiero correspondiente.

---

## RN-082 — Roles que respaldan perfiles comerciales

Todo Seller corresponde a un User con rol `SELLER` y todo Customer a un User con rol `CUSTOMER`. La creación de rol y perfil es atómica. No se retira un rol mientras exista el perfil que depende de él; inactivar un perfil no crea una entidad ni un rol adicional. ADMIN continúa siendo solo un rol de User.

---

## RN-083 — Estabilidad histórica de inventario

Una Account comercializable conserva al menos un Profile y no puede quedar sin el último perfil requerido por una eliminación, desvinculación o inactivación incompatible. Tras la primera asignación histórica, no cambia la Platform de Account ni la Account propietaria de Profile. Las operaciones administrativas entran por la frontera de escritura de inventario: bloquean primero las Accounts raíz por UUID, después Profiles, historial y Subscriptions afectados, y solo entonces emiten DML. Primero deben liberar el recurso mediante el proceso de negocio si existe una asignación incompatible.

---

## RN-084 — Confirmación de orden construida

Una Order PENDING puede construirse sin ítems, pero solo pasa a CONFIRMED si tiene al menos un OrderItem. Cada línea origina exactamente `quantity` Subscriptions y asignaciones iniciales. La comprobación, el bloqueo, el débito y la entrega completa ocurren en una sola transacción; si falta cualquier requisito para una unidad, no se confirma ni queda efecto parcial.

---

## RN-085 — Precio determinado al confirmar

El precio snapshot se determina al confirmar. La transacción valida PriceList activa para el rol aplicable, PriceListItem de la ProductOffer, intervalo vigente, máxima prioridad inequívoca, moneda e importe. Bloquea y revalida la configuración de precio; modificaciones posteriores no cambian la Order confirmada.

---

## RN-086 — Autorización de movimientos de Wallet

TOP_UP y ADJUSTMENT requieren User ADMIN activo. PURCHASE_DEBIT requiere actor autorizado para el Seller buyer y REVERSAL actor autorizado para revertir. La existencia del User se protege referencialmente, pero rol, estado y autorización contextual se validan en la transacción; CUSTOMER no puede efectuar ajustes administrativos.

---

## RN-087 — Modalidad comercial congelada y asignación compatible

ProductOffer define la modalidad comercial fuente de verdad. Al crear y confirmar OrderItem debe conservar exactamente esa modalidad en su snapshot. Desde la primera referencia de una oferta, incluso en una Order PENDING, su modalidad no cambia; una modalidad nueva requiere otra ProductOffer. Toda asignación de ResourceAssignmentHistory consulta ese snapshot: PROFILE exige Profile y FULL_ACCOUNT exige Account. La validación incluye Platform y se aplica tanto a asignación inicial como a garantía; un reemplazo no convierte la modalidad del servicio.

---

## RN-088 — Alta atómica de Seller y Wallet

Todo Seller persistido tiene exactamente una Wallet. El alta autorizada crea en una sola transacción User, rol SELLER, Seller y Wallet con saldo inicial cero, sin movimiento TOP_UP ficticio. La Wallet no puede trasladarse ni eliminarse mientras exista el Seller, y un fallo en cualquier parte revierte toda el alta.

---

## RN-089 — Perfil mínimo por Account

Toda Account confirmada al COMMIT tiene al menos un Profile asociado. La condición no exige que esté AVAILABLE: los estados y asignaciones determinan por separado la elegibilidad comercial. Alta, eliminación y traslado de Profiles solo entran por la frontera PostgreSQL autorizada, que bloquea las Accounts involucradas antes del DML hijo y verifica el mínimo de manera diferida al COMMIT; una Account administrativa puede tener credenciales pendientes, pero no carecer de Profiles.

---

## RN-090 — Secuencia contable por Wallet

Cada WalletTransaction recibe una secuencia contable única y estrictamente creciente dentro de su Wallet. El ledger la asigna bajo bloqueo de Wallet junto con previousBalance, resultingBalance, balance y el siguiente número de secuencia. occurredAt es auditivo y no sustituye el orden contable. Rollback revierte en conjunto movimiento, saldo y contador.

---

## RN-091 — Identidad, credencial y snapshot de contacto

User contiene nombre vigente obligatorio, email obligatorio/único y teléfono opcional. `passwordHash` es hash unidireccional adaptativo; nunca se persiste contraseña en claro. Order toma buyer y beneficiary desde User al crearla y los refresca bajo bloqueo al confirmar; desde `CONFIRMED` los snapshots son inmutables. Cambios posteriores de User no alteran Orders históricas. CU-001 no requiere recuperación, MFA, sesión persistida ni refresh token.

---

## RN-092 — Moneda única configurable

Cada instalación usa un código ISO 4217 fijado por configuración de despliegue antes de crear Wallet. Ese mismo código se conserva en Wallet, Order, OrderItem, PriceListItem y WalletTransaction. No hay cambio por Seller, multidivisa ni conversión automática. Wallet comienza con saldo cero y no genera recarga ficticia. La falta del valor bloquea operar, no diseñar las tablas.

---

## RN-093 — Contrato y acceso a credenciales de streaming

Account puede existir con credenciales pendientes. Para ser elegible requiere `login_identifier`, `secret_payload` cifrado y metadatos no secretos exigidos por el contrato de esa Platform. `credential_metadata` es JSONB opcional y solo contiene información no secreta; proveedor y aplicación validan el contrato antes de publicar inventario. El ciphertext usa envelope con versión/suite, ID de clave, nonce, ciphertext y tag; la clave vive fuera de PostgreSQL. Admin lee todas las credenciales; Seller solo las de sus servicios y Customer solo las de sus servicios activos. Cada lectura se audita y ningún secreto pasa a logs/Eventos.

---

## RN-094 — Cantidad y confirmación completa

`OrderItem.quantity` es entero positivo de unidades con iguales oferta, precio, moneda, modalidad y duración. En confirmación cada unidad genera exactamente una Subscription y una asignación inicial en la misma transacción. Si falla saldo, precio, credencial, compatibilidad o inventario para una unidad, se revierte todo; Order sigue `PENDING` sin débito ni asignación parcial. Una línea con `previousSubscriptionId` debe tener `quantity = 1`; varios servicios previos requieren líneas separadas. OrderItem conserva el objetivo y Subscription confirmada copia el vínculo.

---

## RN-095 — Vigencia y estado derivado de Subscription

La duración vive como días enteros positivos en OrderItem. Subscription persiste `startDate` y `expirationDate`; el estado se calcula por fecha de negocio: `ACTIVE` dentro de `[startDate, expirationDate)` y `EXPIRED` desde `expirationDate`. No se persiste estado, no existe `PENDING → ACTIVE` ni job de expiración. `store.business_timezone` determina `businessDate`. Vencimiento no libera inventario. No hay cancelación anticipada de Subscription confirmada en el MVP.

---

## RN-096 — Definición de métricas y fechas

Los reportes consultan Orders confirmadas por `confirmedAt`, WalletTransaction por `occurredAt` y próximas expiraciones por intervalo explícito de `expirationDate`; su estado de servicio se calcula por RN-027. Venta Seller = Order `CONFIRMED` con Customer beneficiary relacionado mediante SellerCustomer; compra Seller = Order `CONFIRMED` cuyo buyer es ese Seller. Valor bruto = `sum(quantity × unitPrice)`, agrupado por moneda, sin coste, reventa, utilidad o reconocimiento contable. “Ingresos operacionales” es solo etiqueta de ese bruto confirmado; movimientos Wallet se presentan aparte. Se implementa como consulta del backend, no como tabla de reportes/materialización.

---

## RN-097 — Enlace y fecha de renovación/continuidad

OrderItem guarda `previousSubscriptionId` para que una Order PENDING conserve el destino y pueda reintentarse. Al confirmar, Subscription copia el ID; la base valida igualdad. El mismo Customer y Product son obligatorios; una línea refiere un período y solo puede existir un sucesor confirmado. Confirmar requiere que `businessDate` haya alcanzado el vencimiento previo; `startDate = max(previous.expirationDate, businessDate)`. El producto controla renovación para cliente; CU-032 permite continuidad con nuevo recurso cuando corresponda. CU-031/032 con Customer como comprador directo quedan en fase futura.

---

## RN-098 — Ámbito de SellerCustomer y roles

Los códigos `ADMIN`, `SELLER`, `CUSTOMER` son cerrados; Admin asigna roles, sin catálogo dinámico de permisos. Seller vincula/inactiva/reactiva sus relaciones Customer; no crea perfil Customer. SellerCustomer debe estar ACTIVE al crear Order para Customer. Seller consulta solo sus Orders, clientes y servicios originados por sus compras; Customer solo sus servicios; Admin tiene ámbito operativo general. Toda escritura de relación se audita.

---

## RN-099 — Auditoría append-only

AuditEvent registra actor User/System, acción, entidad/ID, instante UTC, cambios no sensibles, razón/contexto seguro y request ID opcional. Cubre cambios de identidad/roles, precios/catálogo, credenciales, cancelación de Order, renovaciones, reposiciones, liberaciones y operaciones administrativas. Cada lectura de credencial genera evento sin incluir el secreto. WalletTransaction es el registro canónico de recargas, débitos, ajustes y reversos, con importe, saldo, secuencia, actor e instante; esos datos no se duplican en AuditEvent. ResourceAssignmentHistory es canónico para recurso, secuencia y liberación; AuditEvent aporta contexto transversal. Ningún registro contiene password, hash, token, clave, secreto ni ciphertext. Los eventos son append-only para el rol de aplicación.