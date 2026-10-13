# Casos de Uso

Los casos de uso representan las principales acciones que los actores
pueden realizar dentro del sistema.

---

# 1. Identity

## CU-001 — Iniciar sesión

**Actor:** Admin, Seller, Customer

El usuario inicia sesión utilizando sus credenciales.

### Resultado

El sistema autentica al usuario y permite acceder a las funcionalidades
correspondientes a su rol.

---

## CU-002 — Gestionar usuarios

**Actor:** Admin

El Admin puede:

- Crear usuarios.
- Consultar usuarios.
- Actualizar usuarios.
- Desactivar usuarios.
- Gestionar roles.

---

# 2. Seller

## CU-003 — Crear Seller

**Actor:** Admin

El Admin registra un nuevo Seller.

La operación crea atómicamente User, rol `SELLER`, perfil Seller y Wallet con saldo inicial cero, sin generar una recarga ficticia. Si falla cualquier paso, se revierte el alta completa.

### Resultado

El Seller queda habilitado para utilizar la plataforma según
los permisos correspondientes.

---

## CU-004 — Consultar Seller

**Actor:** Admin

El Admin puede consultar información del Seller.

Debe poder visualizar:

- Datos básicos.
- Estado derivado: `NOT_REVERSED` o `REVERSED` para movimientos originales, y `REVERSAL` para la fila compensatoria. No existe estado mutable en WalletTransaction.
- Wallet.
- Historial de compras.
- Clientes.
- Servicios.

---

## CU-005 — Desactivar Seller

**Actor:** Admin

El Admin puede desactivar un Seller.

La desactivación no debe eliminar el historial del Seller.

---

# 3. Wallet

## CU-006 — Consultar saldo

**Actor:** Seller, Admin

El usuario consulta el saldo actual de una Wallet.

---

## CU-007 — Recargar Wallet

**Actor:** Admin

El Admin agrega saldo a la Wallet de un Seller.

### Flujo

1. Seleccionar Seller.
2. Consultar Wallet.
3. Introducir monto.
4. Confirmar operación.
5. Registrar movimiento.
6. Actualizar saldo.

### Resultado

La Wallet aumenta su saldo y queda registrada la transacción.

---

## CU-008 — Consultar movimientos de Wallet

**Actor:** Seller, Admin

El usuario consulta el historial de movimientos de la Wallet.

Debe poder identificar:

- Fecha.
- Tipo.
- Monto.
- Saldo anterior.
- Saldo posterior.
- Referencia.
- Estado.

---

# 4. Platform

## CU-009 — Crear Platform

**Actor:** Admin

El Admin registra una plataforma de streaming.

Ejemplos:

- Netflix
- Disney+
- Spotify

---

## CU-010 — Gestionar Platform

**Actor:** Admin

El Admin puede:

- Consultar plataformas.
- Actualizar plataformas.
- Activar plataformas.
- Desactivar plataformas.

---

# 5. Inventory

## CU-011 — Registrar Account

**Actor:** Admin

El Admin registra una Account perteneciente a una Platform.

La misma transacción registra al menos un Profile inicial. Una Account sin Profile no puede confirmar, aunque sus credenciales de streaming estén pendientes de completar.

---

## CU-012 — Registrar Profile

**Actor:** Admin

El Admin registra los Profiles pertenecientes a una Account.

---

## CU-013 — Consultar disponibilidad

**Actor:** Admin, Seller

El usuario consulta qué inventario está disponible para venta.

Debe poder identificar:

- Platform.
- Account.
- Profiles disponibles.
- Modalidad de venta.
- Estado.
- Stock derivado para nuevas asignaciones, según la modalidad.
- Disponibilidad derivada de Account (`AVAILABLE`, `PARTIALLY_ASSIGNED`, `FULLY_ASSIGNED`, `INACTIVE`) y Profile (`AVAILABLE`, `ASSIGNED`, `BLOCKED_BY_FULL_ACCOUNT`, `INACTIVE`) cuando corresponda.

---

# 6. Catalog

## CU-014 — Crear Product

**Actor:** Admin

El Admin registra un producto comercializable.

Al crearlo o editarlo, configura explícitamente si el producto es renovable para cliente y si es renovable internamente. Ambas configuraciones son independientes y no se deducen de Account, Profile o historial.

---

## CU-015 — Crear oferta

**Actor:** Admin

El Admin define una oferta para un Product.

Una oferta puede representar una modalidad como:

- Venta de Profile.
- Venta de Account completa.

---

## CU-016 — Configurar precio

**Actor:** Admin

El Admin configura el precio de una oferta para una determinada
lista de precios.

Como mínimo existen:

- Seller Price.
- Customer Price.

---

## CU-017 — Consultar catálogo

**Actor:** Seller, Customer

El usuario consulta los productos y ofertas disponibles según
su tipo de usuario.

---

# 7. Store

## CU-018 — Comprar Profile

**Actor:** Seller

El Seller compra un Profile disponible.

La cantidad de cada línea es un entero positivo. Cada unidad requiere un Profile independiente, salvo una renovación permitida que conserve el recurso previo.

### Flujo

1. Consultar catálogo.
2. Seleccionar oferta.
3. Seleccionar Profile.
4. Validar que la ProductOffer seleccionada sea modalidad `PROFILE`.
5. Consultar precio Seller.
6. Validar Wallet.
7. Crear Order.
8. Debitar Wallet.
9. Asignar Profile.
10. Registrar duración y crear Subscription para el período de servicio.
11. Confirmar operación.

La confirmación crea exactamente una Subscription por unidad de `quantity`, con asignación inicial para cada una. Si saldo, precio o inventario no alcanza para todas, se revierte todo; la Order permanece `PENDING` sin débito ni asignación parcial.

### Resultado

El Profile queda asignado al cliente correspondiente y se genera
la Subscription.

---

## CU-019 — Comprar Account completa

**Actor:** Seller

El Seller compra una Account completa.

La cantidad de cada línea es un entero positivo y cada unidad requiere una Account elegible independiente.

### Validaciones

- La ProductOffer debe ser modalidad `FULL_ACCOUNT`.
- La Account debe estar disponible.
- Ningún Profile puede estar previamente asignado.
- El Seller debe tener saldo suficiente.

### Flujo

1. Seleccionar Account.
2. Consultar precio Seller.
3. Validar disponibilidad.
4. Validar Wallet.
5. Crear Order.
6. Debitar Wallet.
7. Asignar Account completa.
8. Bloquear Profiles.
9. Registrar duración y crear Subscription para el período de servicio.
10. Confirmar operación.

La confirmación crea todas las Subscriptions y asignaciones iniciales en la misma transacción. Si no se pueden cubrir todas las unidades, no se debita ni se confirma parcialmente.

---

## CU-020 — Consultar historial de compras

**Actor:** Seller, Customer, Admin

El usuario consulta las Orders correspondientes a sus permisos.

---

# 8. Subscription

## CU-021 — Crear Subscription

**Actor:** Sistema

Después de una venta confirmada se crea una Subscription cuando
el producto corresponde a un servicio con duración. La misma transacción crea su `INITIAL_ASSIGNMENT` de secuencia 1; el constraint trigger diferido no permite confirmar la Subscription si esa asignación válida falta. La operación usa la frontera de inventario, que bloquea las Accounts raíz antes de modificar Profiles o historial.

---

## CU-022 — Consultar servicios activos

**Actor:** Seller, Customer

El usuario consulta sus servicios actualmente activos.

---

## CU-023 — Consultar expiraciones

**Actor:** Seller, Admin

El usuario consulta servicios próximos a expirar.

---

## CU-024 — Expirar Subscription

**Actor:** Sistema

Cuando una Subscription alcanza su fecha de expiración,
deja de considerarse activa.

La expiración no elimina el Account o Profile asociado, no lo devuelve automáticamente al stock ni inicia acciones de reacondicionamiento.

---

# 9. Customer

## CU-025 — Comprar servicio directamente (fase futura)

**Actor:** Customer — capacidad prevista para una fase posterior

El Customer seleccionará un producto y realizará el pago mediante
el gateway que se defina.

### Flujo general

1. Consultar catálogo.
2. Seleccionar producto.
3. Seleccionar oferta.
4. Obtener precio Customer.
5. Crear intención de pago.
6. Procesar pago.
7. Confirmar pago.
8. Crear Order.
9. Asignar inventario.
10. Crear Subscription.

---

## CU-026 — Consultar servicios propios

**Actor:** Customer

El Customer puede consultar:

- Servicios activos.
- Plataforma.
- Fecha de inicio.
- Fecha de expiración.
- Estado.

---

# 10. Dashboard Seller

## CU-027 — Consultar Dashboard

**Actor:** Seller

El Seller puede consultar:

- Saldo de Wallet.
- Ventas del período.
- Compras.
- Clientes.
- Servicios activos.
- Próximas expiraciones.
- Historial financiero.

El período de consulta es un intervalo de fechas explícito y se filtra por `confirmedAt` para órdenes y por `expirationDate` para vencimientos. “Ventas” cuenta Orders `CONFIRMED` con beneficiary Customer relacionado al Seller; “compras” cuenta sus Orders `CONFIRMED` como buyer. Importes brutos son `sum(quantity × unitPrice)`, agrupados por moneda; no representan margen ni utilidad.

---

# 11. Dashboard Admin

## CU-028 — Consultar Dashboard Administrativo

**Actor:** Admin

El Admin puede consultar métricas generales de la operación.

Como mínimo:

- Ventas.
- Compras.
- Sellers.
- Inventario.
- Servicios activos.
- Servicios próximos a expirar.
- Movimientos de Wallet.
- Ingresos.

Los indicadores de ventas/compras y valores brutos usan solo Orders `CONFIRMED` cuya fecha `confirmedAt` cae en el intervalo pedido; monto bruto es suma de `quantity × unitPrice`, sin descontar costos ni presentar utilidad. “Ingresos operacionales” es etiqueta de ese valor bruto confirmado, no ingreso contable; reversiones y ajustes se presentan aparte desde WalletTransaction. Fechas futuras/próximas a vencer se consultan en un rango explícito y solo incluyen Subscription `ACTIVE`.

---

# 12. Reportes

## CU-029 — Consultar ventas

**Actor:** Admin, Seller

El sistema permite consultar ventas según los permisos
correspondientes.

Para este caso, una venta de Seller es una Order `CONFIRMED` en que el beneficiario es Customer de su relación SellerCustomer; se agrupa por `confirmedAt` y se informa importe bruto por moneda. Las compras del Seller son Orders `CONFIRMED` cuyo buyer es ese Seller. El MVP no conoce precio de reventa ni margen del Seller.

---

## CU-030 — Consultar movimientos financieros

**Actor:** Admin

El Admin puede consultar los movimientos financieros
relacionados con las Wallets.

El reporte filtra por `occurredAt`, muestra delta firmado y saldo posterior, agrupa por moneda/tipo y conserva cada `REVERSAL` como movimiento separado; no oculta ni reescribe el original.

---

# 13. Renovaciones y Control de Cuentas

## CU-031 — Renovar servicio para cliente

**Actor:** Seller (MVP); Customer como comprador directo (fase futura)

El Seller puede preparar una Order `PENDING` para un servicio vencido o próximo a vencer. La línea lleva `previousSubscriptionId` y `quantity = 1`; preparar o mantener pendiente la orden no cobra ni reserva inventario.

### Confirmación y resultado

- El Product debe permitir renovación para cliente; el beneficiario y Product deben coincidir con la Subscription anterior.
- La orden solo se confirma cuando `businessDate >= previousSubscription.expirationDate`. Si se intenta antes, permanece `PENDING`.
- `startDate = max(previousSubscription.expirationDate, businessDate)`. La fecha de inicio queda en el día de confirmación, sin servicio retroactivo ni período programado futuro.
- Se crea una nueva OrderItem y Subscription enlazadas a la Subscription previa. Un período admite como máximo un sucesor confirmado.
- Si se conserva Account/Profile, la asignación anterior se libera por `RENEWAL_TRANSFER` y se crea la asignación inicial de la nueva Subscription en una transacción, sin consumo adicional de stock.
- Si se usa otro recurso, debe estar disponible y consume inventario. El recurso anterior permanece retenido hasta su reacondicionamiento y publicación manual.
- Saldo, precio, elegibilidad o asignación insuficientes revierten el intento completo; no queda débito, Subscription ni asignación parcial, y Order sigue `PENDING`.

## CU-032 — Asignar nuevo recurso para continuidad

**Actor:** Seller (MVP); Customer como comprador directo (fase futura)

Cuando el producto no admite renovación para cliente o no se conservará el recurso, Seller prepara una Order con una línea de `quantity = 1`, el nuevo recurso y `previousSubscriptionId`. La nueva asignación requiere inventario disponible y el período anterior como referencia de continuidad.

La confirmación solo se permite en o después del vencimiento anterior. `startDate = max(previousSubscription.expirationDate, businessDate)`. Se crea el nuevo período y su asignación inicial atómicamente; se aplica toda la validación de saldo, precio y disponibilidad, sin reserva anticipada ni cumplimiento parcial. El recurso anterior no vuelve automáticamente al stock y debe liberarse/publicarse después del reacondicionamiento administrativo.

---
## CU-033 — Consultar Control de Cuentas

**Actor:** Admin

El administrador consulta el estado operativo e histórico de recursos asignados.

Debe poder consultar, cuando aplique:

- Platform, Account y Profile.
- Número de pedido, buyer, beneficiary, seller de origen, nombre, correo y teléfono cuando aplique.
- Producto, Subscription, inicio, vencimiento, duración, días restantes y estado.
- Modalidad de venta e información de renovación, garantía/reemplazo y datos operativos de acceso cuando permisos lo permitan.

Este caso de uso describe una consulta de inventario + asignaciones + suscripciones; no define una entidad nueva.

---

## CU-034 — Volver a cargar recurso reacondicionado

**Actor:** Admin

Después de un proceso manual externo de reacondicionamiento, el administrador vuelve a cargar/publicar un Account o Profile cuyo Product está configurado como renovable internamente.

### Resultado

El recurso vuelve a formar parte del inventario disponible por la nueva carga/publicación administrativa, no porque la plataforma haya detectado el vencimiento. Al publicarlo, Admin libera la RAH retenida con motivo `MANUAL_RECONDITIONING`, actor y hora, y habilita/valida las credenciales. Esta operación no se confunde con una renovación del cliente.

---

## CU-035 — Reponer recurso por garantía

**Actor:** Admin

Durante un período contratado vigente, el recurso falla y el negocio entrega otro recurso para solucionar el problema. Para un perfil, el reemplazo puede pertenecer a la misma cuenta o a otra cuenta.

### Resultado

La operación se trata como garantía/reposición, no como renovación ni nueva compra. La Subscription existente conserva su fecha de vencimiento y días restantes; se agrega una asignación histórica de reemplazo y el recurso defectuoso queda no disponible para revisión. Puede haber múltiples reposiciones durante el período si corresponden a garantía.

La operación entra por la misma frontera de inventario: bloquea todas las Accounts raíz afectadas en orden UUID, después Profiles, historial y Subscription, libera la asignación vigente y agrega `WARRANTY_REPLACEMENT` en una única transacción. Un rollback no deja una liberación o reemplazo parcial.

La asignación conserva actor, fecha y descripción de la falla. La fila previa identifica el recurso fallido y registra su liberación por `WARRANTY_REPLACEMENT`; la nueva fila identifica el reemplazo y conserva actor, fecha y descripción de la falla.

## CU-036 — Vincular o inactivar Customer de Seller

**Actor:** Seller, Admin

Seller vincula a su gestión un Customer existente; Admin puede hacerlo en tareas administrativas. Seller puede inactivar/reactivar solo su relación comercial. La acción no crea un User Customer y no elimina órdenes o servicios históricos. No se crea la relación automáticamente durante una compra.

### Resultado

Queda una relación `SellerCustomer` explícita y auditable. Solo una relación activa permite crear nuevas Orders para ese Customer.

---

# Dependencias principales

```text
    Crear Seller + Wallet inicial
    ↓
Recargar Wallet
    ↓
Consultar catálogo
    ↓
Comprar Profile / Account
    ↓
Crear Order
    ↓
Asignar Inventory
    ↓
Crear Subscription
    ↓
Consultar servicios
