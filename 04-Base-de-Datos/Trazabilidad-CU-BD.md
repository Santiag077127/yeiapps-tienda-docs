# Trazabilidad CU → modelo PostgreSQL

Los Casos de Uso son la fuente funcional mientras no exista backlog formal. La tabla enlaza cada caso con reglas (`RN`), entidades/atributos, operación y estructuras físicas documentadas. No implica que las tablas existan: el changelog de tablas sigue vacío.

| CU | Regla(s) | Entidades y datos relevantes | Operación y estructura documental |
|---|---|---|---|
| CU-001 Iniciar sesión | RN-001, RN-002, RN-091 | User: email, passwordHash, status; UserRole | Autenticar y autorizar por rol; `identity.app_user`, `identity.user_role`. Sin persistencia de sesión/refresh token MVP. |
| CU-002 Gestionar usuarios | RN-001, RN-033, RN-082, RN-091 | User, UserRole, Seller, Customer | Crear/consultar/editar/inactivar y asignar roles; `identity.app_user`, `user_role`, `seller`, `customer`, `audit_event`. Códigos de rol fijos. |
| CU-003 Crear Seller | RN-003, RN-088 | User, UserRole, Seller, Wallet | Alta atómica con Wallet saldo 0; `identity.*`, `wallet.wallet`, `audit_event`. |
| CU-004 Consultar Seller | RN-033, RN-081 | Seller, User, Wallet, Order, SellerCustomer, Subscription | Consulta administrativa; `identity.*`, `wallet.wallet`, `sales.orders`, `sales.subscription`. |
| CU-005 Desactivar Seller | RN-071, RN-073, RN-033 | Seller.status, User.status | Transición sin cascada ni borrado; `identity.seller`, `audit_event`. |
| CU-006 Consultar saldo | RN-003, RN-081, RN-090 | Wallet.balance, currencyCode, sequence; WalletTransaction.resultingBalance | Consulta de saldo y moneda; `wallet.wallet`, `wallet.wallet_transaction`. |
| CU-007 Recargar Wallet | RN-004, RN-006, RN-075, RN-086, RN-090 | Wallet, WalletTransaction (type, amount, saldos, actor, secuencia) | `TOP_UP` transaccional por Admin; `wallet.wallet`, `wallet.wallet_transaction`, `identity.audit_event`. |
| CU-008 Consultar movimientos | RN-006, RN-033, RN-076, RN-090, RN-092 | WalletTransaction: ocurrido, tipo, delta, saldos, referencias; estado derivado de reverso | Consulta cronológica/contable; `wallet.wallet_transaction`; `REVERSED`/`NOT_REVERSED` derivado de `REVERSAL`, sin columna status. |
| CU-009 Crear Platform | RN-007, RN-033 | Platform.name/status | Crear plataforma; `catalog.platform`, `identity.audit_event`. |
| CU-010 Gestionar Platform | RN-007, RN-071, RN-073 | Platform.status | Consultar/editar/inactivar/reactivar; `catalog.platform`, `audit_event`. |
| CU-011 Registrar Account | RN-008, RN-032, RN-034, RN-089, RN-093 | Account.platformId, loginIdentifier, secretPayload, credentialMetadata, isActive; Profile inicial | Crear Account + ≥1 Profile atómicamente; `inventory.account`, `inventory.profile`; elegibilidad desde vista. |
| CU-012 Registrar Profile | RN-009, RN-010, RN-089, RN-093 | Profile.accountId, profileIdentifier, isActive | Operación bajo frontera autorizada; `inventory.profile`, `inventory.account_availability`, `profile_availability`, `audit_event`. |
| CU-013 Consultar disponibilidad | RN-034, RN-038, RN-044, RN-062, RN-063 | Account/Profile isActive, credenciales, ResourceAssignmentHistory | Proyección por modalidad; vistas `inventory.account_availability`, `inventory.profile_availability` y tabla de historial. |
| CU-014 Crear Product | RN-016, RN-046, RN-047, RN-070 | Product.platformId, canRenewForCustomer, internallyRenewable, status | Alta/edición catálogo; `catalog.product`, `audit_event`. |
| CU-015 Crear oferta | RN-011, RN-017, RN-087 | ProductOffer.productId, saleMode, status | Oferta `PROFILE` o `FULL_ACCOUNT`; `catalog.product_offer`. |
| CU-016 Configurar precio | RN-018, RN-019, RN-020, RN-074, RN-085, RN-092 | PriceList.targetRole/status; PriceListItem amount, currency, validity, priority | Publicar y resolver precio no ambiguo; `catalog.price_list`, `price_list_item`. |
| CU-017 Consultar catálogo | RN-016–020 | Product, ProductOffer, PriceList/Item | Seller consulta en MVP; Customer orientado a compra queda futuro; tablas `catalog.*`. |
| CU-018 Comprar Profile | RN-012, RN-021–024, RN-034–038, RN-069, RN-084, RN-085, RN-087, RN-094 | Order snapshots/beneficiary/currency/status; OrderItem quantity/price/duration/mode; Wallet, Subscription, RAH | Confirmación atómica e idempotente; `sales.orders`, `order_item`, `subscription`, `inventory.resource_assignment_history`, `wallet.*`, `audit_event`. |
| CU-019 Comprar Account completa | RN-013–015, RN-021–024, RN-069, RN-084, RN-087, RN-094 | Mismos datos de venta; Account elegible y Profiles bloqueados por vista | Confirmar `FULL_ACCOUNT` todo o nada; mismas tablas que CU-018 más `inventory.account/profile`. |
| CU-020 Consultar historial de compras | RN-022, RN-030, RN-067 | Order/OrderItem snapshots, buyer, beneficiary, fechas | Consulta por ámbito; `sales.orders`, `sales.order_item`. |
| CU-021 Crear Subscription | RN-025–027, RN-064, RN-066, RN-095 | Subscription: OrderItem, startDate, expirationDate, previousSubscriptionId; RAH initial assignment | Al confirmar cada unidad se crea Subscription y asignación inicial; estado ACTIVE/EXPIRED derivado, sin estado persistido. |
| CU-022 Consultar servicios activos | RN-025, RN-027, RN-043, RN-064, RN-095 | Subscription dates; Order beneficiary; RAH retenido | Consulta Seller de sus Orders y Customer de sus propios servicios usando intervalo `[startDate, expirationDate)` y businessDate; sin status column. |
| CU-023 Consultar expiraciones | RN-026, RN-027, RN-043, RN-095, RN-096 | Subscription.expirationDate | Rango de fechas de negocio y consulta `sales.subscription` por índice `expiration_date`; estado derivado. |
| CU-024 Expirar Subscription | RN-026, RN-027, RN-064, RN-095 | Subscription.startDate/expirationDate | Al llegar al límite, las consultas la consideran EXPIRED por fecha; no hay job ni UPDATE y no se libera inventario. |
| CU-025 Compra directa Customer (futuro) | RN-029 | Payment futuro, Order y Subscription | Fuera de MVP: sin tabla `Payment` ni cambios; gateway y estados pendientes de decisión. |
| CU-026 Consultar servicios propios | RN-028, RN-043 | Customer User/profile; Order beneficiary; Subscription y RAH | Acceso de solo lectura a servicios propios; `identity.customer`, `sales.*`, `inventory.resource_assignment_history`. |
| CU-027 Dashboard Seller | RN-043, RN-096 | Order confirmedAt, beneficiary, Wallet, Subscription | Período explícito; métricas propias y bruto por moneda; consultas de `sales.*`, `wallet.*`. |
| CU-028 Dashboard Admin | RN-033, RN-096 | Order, Seller/Customer, inventario, Subscription, WalletTransaction | Totales operacionales; joins/vistas, auditoría de acciones sensibles. |
| CU-029 Consultar ventas | RN-021, RN-022, RN-096 | Order.status/confirmedAt, beneficiaryCustomer, OrderItem quantity/unitPrice | Seller sales = Orders confirmadas para Customer; gross = q×precio; `sales.orders`, `order_item`. |
| CU-030 Consultar movimientos financieros | RN-006, RN-033, RN-076, RN-090, RN-096 | WalletTransaction ocurrido/tipo/importe/moneda | Reporte de ledger por rango y tipo, reverso separado; `wallet.wallet_transaction`. |
| CU-031 Renovar servicio para cliente | RN-039–041, RN-046–048, RN-065, RN-097 | OrderItem.previousSubscriptionId/quantity; Subscription.previousSubscriptionId/fechas; RAH | Seller puede preparar una línea PENDING `quantity=1`; solo confirma al alcanzar vencimiento previo. Se copia el vínculo a Subscription, con un único sucesor confirmado. Recurso previo se transfiere atómicamente si se conserva; sin pre-reserva. |
| CU-032 Asignar nuevo recurso para continuidad | RN-041, RN-064–069, RN-097 | OrderItem.previousSubscriptionId; Subscription dates; RAH | Confirma en/después del vencimiento previo, asigna recurso disponible y retiene el anterior hasta republicación manual; transacción todo o nada. |
| CU-033 Control de Cuentas | RN-043, RN-056, RN-096 | Snapshots, Subscription, OrderItem, User data, RAH, credenciales permitidas | Consulta operacional con alcance de acceso; `sales.*`, `identity.*`, `inventory.*`; lectura de secreto auditable. |
| CU-034 Volver a cargar recurso reacondicionado | RN-045, RN-049, RN-058, RN-059, RN-066, RN-093 | Account/Profile.isActive; RAH releasedAt/releasedBy/releaseReason; credenciales | Admin libera con `MANUAL_RECONDITIONING`, configura credenciales/habilitación y publica; nunca retorno automático por expiración. |
| CU-035 Reponer recurso por garantía | RN-050–057, RN-066, RN-079, RN-097 | Subscription, RAH reason/sequence/actor/failure/release | Liberar fila vigente + crear reemplazo atómicamente; `inventory.resource_assignment_history`, disponibilidad/vistas, `audit_event`. |
| CU-036 Vincular Customer existente a Seller | RN-033, RN-072, RN-098 | SellerCustomer seller/customer/status; actor | Crear/inactivar/reactivar relación explícita; `identity.seller_customer`, `audit_event`; no se crea Customer User. |

## Lectura de la matriz

- Los estados persistidos son solo los enumerados en el modelo físico. Estado de reverso y disponibilidad de inventario son derivados.
- Los snapshots de Order y OrderItem se congelan en confirmación. Identidad vigente no reescribe ventas anteriores.
- “Ingresos” reportados por el MVP son valor bruto operacional de Orders confirmadas; no utilidad ni contabilidad financiera.
- CU-025 y sus estructuras quedan fuera del esquema actual.
