# Modelo físico PostgreSQL

Este documento congela las decisiones de diseño físico del MVP. Es una especificación documental: no contiene SQL, migraciones ni código ejecutable.

## Esquemas y convenciones

```text
identity.app_user
identity.user_role
identity.audit_event
identity.seller
identity.customer
identity.seller_customer
catalog.platform
catalog.product
catalog.product_offer
catalog.price_list
catalog.price_list_item
inventory.account
inventory.profile
inventory.resource_assignment_history
sales.orders
sales.order_item
sales.subscription
wallet.wallet
wallet.wallet_transaction
```

El inventario canónico contiene **19 tablas**: seis en `identity`, cinco en `catalog`, tres en `inventory`, tres en `sales` y dos en `wallet`. `identity.audit_event` es append-only y cubre auditoría transversal, incluidas lecturas de credenciales. Esta lista es la fuente del alcance físico.

`public` no contiene tablas de negocio. Todas las referencias deben estar calificadas por esquema y las FKs históricas usan `ON DELETE RESTRICT` y `ON UPDATE RESTRICT`.

Todas las PK son `uuid`. Los eventos usan `timestamptz`; las fechas de Subscription usan `date`. Los importes usan 
`numeric(19,4)` y las monedas `char(3)` ISO 4217; no se usan flotantes. El redondeo comercial final es a dos decimales. Un `CHECK` de moneda verifica tres letras mayúsculas, pero la pertenencia al catálogo ISO se valida en aplicación.

`store.currency_code` es una configuración externa única por instalación, con código ISO 4217 validado. Wallet nueva y todas las operaciones financieras/comerciales de esa instalación usan ese valor; no existe selección multi-moneda ni conversión. `store.business_timezone` configura el día de negocio usado por expiración y reportes diarios. Los valores concretos no se fijan en el modelo ni en los changelogs.

Los estados persistidos con `CHECK` son: entidades de identidad/catálogo/relación `ACTIVE`/`INACTIVE`; Order `PENDING`/`CONFIRMED`/`CANCELLED`; Account/Profile persisten solo `is_active` y su disponibilidad se deriva en vistas. Subscription no persiste estado: `ACTIVE`/`EXPIRED` se calcula por fechas. WalletTransaction persiste tipos `TOP_UP`, `PURCHASE_DEBIT`, `ADJUSTMENT_CREDIT`, `ADJUSTMENT_DEBIT`, `REVERSAL`, sin estado mutable.

## Normalización y snapshots

- **1FN:** columnas del dominio son escalares y cada movimiento/ítem/asignación ocupa una fila. `credential_metadata` y `AuditEvent.changed_fields/details` son JSONB controlados para metadatos variables/contexto no sensible; no contienen valores core como identidad, precio, cantidad, estado ni relaciones.
- **2FN:** claves primarias simples UUID evitan dependencias parciales. Las relaciones N:N usan `user_role` y `seller_customer`; claves candidatas compuestas protegen las correspondencias SellerCustomer y ProductOffer/Product.
- **3FN:** identidad, perfiles, catálogo, precio, recursos, órdenes, servicios, Wallet y ledger tienen una fuente canónica por entidad. Disponibilidad de Account/Profile y estado de Subscription se calculan; no se duplican como columnas operativas.
- **Snapshots históricos intencionales:** Order conserva buyer/beneficiary desde User; OrderItem conserva Product, oferta, precio, moneda, duración, modalidad y cantidad para que cambios posteriores no reescriban una operación confirmada.
- **Continuidad:** OrderItem.previous_subscription_id guarda la intención aun en una Order PENDING; Subscription.previous_subscription_id registra la relación efectivamente confirmada. La copia es deliberada para preservar tanto borrador reintentable como linaje del período; igualdad se valida en la transacción.
- **Wallet.balance:** saldo materializado para lectura, reconstruible desde movimientos ordenados por `sequence_no`. Solo cambia al insertar WalletTransaction bajo el mismo lock y transacción; el saldo posterior del movimiento más reciente debe coincidir.
- **Moneda:** currency_code se repite en Wallet, Order, OrderItem, PriceListItem y WalletTransaction como unidad/valor histórico comprobable; se valida contra la configuración única de despliegue y entre filas relacionadas al confirmar.

`CHECK`, FK y triggers no crean índices por sí mismos. `jsonb` no reemplaza columnas de identidad, precio o relaciones.
## Columnas, nulabilidad y restricciones

### identity

#### `identity.app_user`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Identificador técnico. |
| name | varchar(200) | No | — | Nombre vigente de la persona. |
| email | varchar(320) | No | Único sin distinción de mayúsculas; formato validado por aplicación | Correo de acceso. |
| phone | varchar(50) | Sí | — | Teléfono opcional normalizado. |
| password_hash | text | No | Nunca texto plano | Hash de autenticación. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado de identidad. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación permitida. |

#### `identity.audit_event`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Evento. |
| actor_type | varchar(10) | No | CHECK `USER`/`SYSTEM` | Origen. |
| actor_user_id | uuid | Sí | FK a `app_user`; requerido para USER y nulo para SYSTEM | Usuario responsable. |
| action_code | varchar(80) | No | — | Acción auditada, incluida lectura de credencial. |
| entity_type | varchar(80) | No | — | Tipo del objeto de negocio. |
| entity_id | uuid | No | Polimórfica, sin FK | ID del objeto auditado. |
| occurred_at | timestamptz | No | — | Momento UTC del evento. |
| changed_fields | jsonb | Sí | Solo before/after no secretos | Campos modificados. |
| details | jsonb | Sí | Nunca secretos, contraseñas, hashes o tokens | Contexto seguro, razón y resultado. |
| request_id | varchar(100) | Sí | — | Correlación con solicitud. |

La tabla es append-only para la aplicación: sin UPDATE ni DELETE. La auditoría detallada de ledger y asignaciones sigue en WalletTransaction y ResourceAssignmentHistory; AuditEvent referencia esos eventos sin duplicar importes ni secretos.

#### `identity.user_role`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| user_id | uuid | No | PK compuesta; FK a `app_user` | Usuario. |
| role_code | varchar(20) | No | PK compuesta; CHECK `ADMIN`, `SELLER`, `CUSTOMER` | Rol asignado. |

#### `identity.seller` y `identity.customer`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Identificador del perfil de negocio. |
| user_id | uuid | No | FK a `app_user`; UNIQUE | Usuario dueño. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado comercial. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

La FK a `app_user` garantiza existencia, no el rol. La garantía principal se ejecuta en la transacción de alta: bloquear User y sus filas `user_role`, crear primero el rol requerido y después el perfil comercial. Triggers de integridad en `seller` y `customer` rechazan INSERT/UPDATE cuyo User no tenga, respectivamente, `SELLER` o `CUSTOMER`; un trigger en `user_role` rechaza retirar ese rol mientras exista el perfil correspondiente. Un User puede conservar varios roles y `ADMIN` sigue siendo solo un rol. No existe estado de rol: una transición administrativa permitida debe inactivar el perfil sin retirar su rol; el rol no se elimina mientras el perfil exista.

#### `identity.seller_customer`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Identificador de la relación. |
| seller_id | uuid | No | FK a `seller`; UNIQUE compuesta con `customer_id` | Seller relacionado. |
| customer_id | uuid | No | FK a `customer`; UNIQUE compuesta con `seller_id` | Customer relacionado. |
| status_code | varchar(20) | No | CHECK `ACTIVE`/`INACTIVE` | Vigencia comercial para nuevas órdenes. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

### catalog

#### `catalog.platform`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Plataforma. |
| name | varchar(150) | No | Índice de consulta | Nombre operativo. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado del catálogo. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

#### `catalog.product`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Producto. |
| platform_id | uuid | No | FK a `platform` | Plataforma compatible. |
| name | varchar(200) | No | Índice de consulta | Nombre comercial. |
| customer_renewable | boolean | No | — | Renovabilidad por Customer. |
| internally_renewable | boolean | No | — | Renovabilidad interna. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado del catálogo. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

#### `catalog.product_offer`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK; UNIQUE candidata con `product_id` | Oferta. |
| product_id | uuid | No | FK a `product`; UNIQUE candidata con `id` | Producto ofrecido. |
| sale_mode | varchar(20) | No | CHECK `PROFILE`/`FULL_ACCOUNT` | Modalidad vendida. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado de la oferta. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

Además de la PK, existe una unicidad parcial de `(product_id, sale_mode)` solo para filas `ACTIVE`: un producto no publica dos ofertas activas de la misma modalidad. Las ofertas inactivas históricas no bloquean una nueva oferta activa.

#### `catalog.price_list`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Lista de precios. |
| name | varchar(150) | No | — | Nombre operativo. |
| target_role | varchar(20) | No | CHECK `SELLER`/`CUSTOMER` | Segmento comercial. |
| status_code | varchar(30) | No | CHECK `ACTIVE`/`INACTIVE` | Estado de publicación. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

Existe una unicidad parcial de `target_role` solo donde la lista está `ACTIVE`: hay como máximo una lista activa para `SELLER` y una para `CUSTOMER`. El cambio de lista activa se realiza en una sola transacción que bloquea y revalida las listas del rol; nunca debe quedar un instante confirmado con dos listas activas.

#### `catalog.price_list_item`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Ítem de precio. |
| price_list_id | uuid | No | FK a `price_list` | Lista propietaria. |
| product_offer_id | uuid | No | FK a `product_offer` | Oferta cotizada. |
| amount | numeric(19,4) | No | CHECK `amount >= 0` | Importe. |
| currency_code | char(3) | No | CHECK de formato ISO | Moneda del precio. |
| valid_from | timestamptz | No | — | Inicio inclusivo. |
| valid_to | timestamptz | Sí | CHECK `valid_to > valid_from` cuando existe | Fin exclusivo. |
| priority | integer | No | CHECK `priority > 0` | Prioridad de precio. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

La vigencia es el intervalo semiabierto `[valid_from, valid_to)`. Una restricción de exclusión `EXCLUDE USING gist`, sobre `price_list_id`, `product_offer_id`, `priority` y el rango temporal, prohíbe todo solapamiento para esa combinación. Requiere la extensión PostgreSQL `btree_gist`. Intervalos consecutivos son válidos; intervalos solapados con prioridades distintas son válidos y se resuelven por la mayor prioridad. Así no existe empate aplicable de igual prioridad.

### inventory

#### `inventory.account`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Cuenta de streaming. |
| platform_id | uuid | No | FK a `platform` | Plataforma de la cuenta. |
| login_identifier | text | Sí | — | Identificador de acceso si se conoce. |
| secret_payload | bytea | Sí | Cifrado antes de persistir | Credencial secreta cifrada. |
| credential_metadata | jsonb | Sí | Sin secreto en claro | Metadatos de la credencial. |
| is_active | boolean | No | DEFAULT true | Habilitación manual; no puede cambiarse si hay asignación vigente. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

Una Account puede existir como borrador sin credencial completa. Para asignación comercial exige `login_identifier`, `secret_payload` cifrado y metadatos requeridos válidos. La clave vive fuera de PostgreSQL. `is_active` es la única habilitación persistida; disponibilidad se deriva en `inventory.account_availability`.

#### `inventory.profile`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Perfil de una Account. |
| account_id | uuid | No | FK a `account`; UNIQUE con `profile_identifier` | Cuenta propietaria. |
| profile_identifier | varchar(100) | No | UNIQUE con `account_id` | Identificador dentro de la cuenta. |
| is_active | boolean | No | DEFAULT true | Habilitación manual para nuevas asignaciones. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación. |

Toda Account persistida debe tener al menos un Profile asociado al `COMMIT`, sin importar que sus credenciales aún sean incompletas. Una Account administrativa incompleta significa credenciales pendientes, no ausencia de Profiles. La creación de Account y Profiles iniciales ocurre en una única transacción; un INSERT aislado de Account falla al `COMMIT`.

`inventory.profile_availability` deriva `INACTIVE` si Account/Profile están deshabilitados o faltan credenciales comerciales, `ASSIGNED` si existe asignación vigente, `BLOCKED_BY_FULL_ACCOUNT` si la Account está asignada completa, y `AVAILABLE` en otro caso elegible. `inventory.account_availability` deriva `INACTIVE`, `AVAILABLE`, `PARTIALLY_ASSIGNED` o `FULLY_ASSIGNED` de `is_active`, credenciales, Profiles elegibles y asignaciones vigentes. Ninguno de esos códigos se persiste ni se acepta como fuente de verdad.

Constraint triggers `AFTER`, `DEFERRABLE INITIALLY DEFERRED`, en Account y Profile verifican al `COMMIT` que cada Account existente tenga al menos un Profile asociado. Cubren INSERT de Account, INSERT/DELETE de Profile y UPDATE de `profile.account_id`: en un traslado se revalidan tanto `OLD.account_id` como `NEW.account_id`. No exigen un Profile `AVAILABLE` ni `ACTIVE`; el mínimo es de asociación y la disponibilidad continúa regulada por estados y asignaciones. Su ejecución diferida consulta el estado final de la transacción: un INSERT aislado de Account y la eliminación o traslado del último Profile fallan al `COMMIT`.

Los triggers de Profile no intentan adquirir tardíamente locks de Account. Toda escritura autorizada entra por la frontera de inventario descrita en «Protocolo global de locks»: bloquea las Accounts raíz antes de emitir `INSERT`, `UPDATE` o `DELETE` de Profile. Si el Profile tiene ResourceAssignmentHistory, su eliminación física está prohibida por FK `RESTRICT` y su `account_id` se vuelve inmutable mediante trigger y privilegios; si aún no tiene historial, puede corregirse solo por esa frontera tras bloquear y revalidar ambas Accounts, sus Profiles y asignaciones vigentes. DELETE de Account no está permitido al rol de aplicación; además, las FK de Profile y el mínimo diferido impiden eliminar una Account conservando integridad.

`account.platform_id` es corregible solo antes de la primera ResourceAssignmentHistory asociada a la Account o a cualquiera de sus Profiles. Desde el primer uso comercial —vigente o liberado— un trigger y los permisos del rol de aplicación rechazan su cambio. Esto evita reinterpretar la Platform de una asignación histórica.

#### `inventory.resource_assignment_history`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Asignación histórica. |
| subscription_id | uuid | No | FK a `sales.subscription` | Período atendido. |
| account_id | uuid | Sí | FK a `account`; XOR con `profile_id` | Recurso FULL_ACCOUNT. |
| profile_id | uuid | Sí | FK a `profile`; XOR con `account_id` | Recurso PROFILE. |
| assigned_at | timestamptz | No | — | Inicio de uso. |
| released_at | timestamptz | Sí | CHECK `released_at > assigned_at` cuando existe | Fin de uso. |
| reason_code | varchar(30) | No | CHECK `INITIAL_ASSIGNMENT`/`WARRANTY_REPLACEMENT` | Motivo cerrado. |
| sequence | integer | No | UNIQUE con `subscription_id`; CHECK `sequence > 0` | Orden del historial. |
| performed_by_user_id | uuid | No | FK a `identity.app_user` | Actor responsable. |
| failure_occurred_at | timestamptz | Sí | Requerido solo para `WARRANTY_REPLACEMENT` | Momento reportado de falla. |
| failure_description | text | Sí | Requerido solo para `WARRANTY_REPLACEMENT`; no incluir secretos | Descripción de falla. |
| released_by_user_id | uuid | Sí | FK a `identity.app_user`; obligatorio junto con `released_at` | Actor de liberación. |
| release_reason_code | varchar(30) | Sí | CHECK de motivos de liberación; obligatorio junto con `released_at` | `WARRANTY_REPLACEMENT`, `RENEWAL_TRANSFER` o `MANUAL_RECONDITIONING`. |

Hay unicidad parcial para una asignación retenida por Subscription, por Profile y por Account cuando `account_id` no es nulo (`released_at IS NULL`). Una exclusión temporal GiST —con `btree_gist` para igualdad UUID— impide solapamientos finitos del mismo Profile o de FULL_ACCOUNT sobre la misma Account; los índices parciales también impiden dos asignaciones abiertas. Las filas PROFILE pueden coexistir en una Account cuando usan Profiles distintos. La incompatibilidad entre FULL_ACCOUNT y sus Profiles exige bloqueo de Account, consulta y revalidación transaccional. `INITIAL_ASSIGNMENT` es la primera asignación; `WARRANTY_REPLACEMENT` exige asignación previa liberada y conserva la vigencia de Subscription.

Un trigger `BEFORE INSERT OR UPDATE` recorre `Subscription → OrderItem → ProductOffer` y bloquea las filas comerciales con `FOR SHARE`. La función autorizada ya bloqueó Account raíz antes del DML. Revalida modo y Platform; exige XOR del recurso. `INITIAL_ASSIGNMENT` usa secuencia 1; `WARRANTY_REPLACEMENT` usa secuencia mayor, requiere asignación anterior liberada y conserva el período. Un cierre se completa una única vez y atómicamente en `released_at`, `released_by_user_id` y `release_reason_code`; un CHECK exige que los tres sean nulos o no nulos juntos, y un trigger impide reabrir o editar el dato histórico. Los motivos permitidos son reposición, transferencia por renovación y reacondicionamiento manual. La validación de modalidad/Platform y la secuencia son intertablas y se aplican en la función/trigger; XOR y FKs protegen la forma básica de la fila.

### sales

#### `sales.orders`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Orden interna. |
| order_number | uuid | No | UNIQUE; inmutable | Número comercial aleatorio. |
| buyer_seller_id | uuid | No | FK a `seller` | Seller comprador. |
| beneficiary_seller_id | uuid | Sí | FK a `identity.seller(id)`; XOR con `beneficiary_customer_id`; debe igualar a buyer cuando existe | Beneficiario Seller. |
| beneficiary_customer_id | uuid | Sí | XOR con `beneficiary_seller_id`; FK compuesta con buyer a `seller_customer` | Beneficiario Customer. |
| buyer_name_snapshot | varchar(200) | No | Inmutable tras confirmación | Nombre del comprador. |
| buyer_email_snapshot | varchar(320) | No | Inmutable tras confirmación | Correo del comprador. |
| buyer_phone_snapshot | varchar(50) | Sí | Inmutable tras confirmación | Teléfono disponible. |
| beneficiary_name_snapshot | varchar(200) | No | Inmutable tras confirmación | Nombre del beneficiario. |
| beneficiary_email_snapshot | varchar(320) | No | Inmutable tras confirmación | Correo del beneficiario. |
| beneficiary_phone_snapshot | varchar(50) | Sí | Inmutable tras confirmación | Teléfono disponible. |
| currency_code | char(3) | No | CHECK de formato ISO; inmutable tras confirmación | Moneda única de la orden. |
| status_code | varchar(30) | No | CHECK de estados Order | Estado comercial. |
| created_at | timestamptz | No | — | Creación. |
| confirmed_at | timestamptz | Sí | Obligatorio si CONFIRMED; nulo en otro caso | Confirmación. |

`buyer_seller_id` y, cuando corresponda, `beneficiary_seller_id` usan FKs `RESTRICT` a `identity.seller(id)`. La relación con Customer se preserva con la FK compuesta `(buyer_seller_id, beneficiary_customer_id)` hacia la clave candidata `(seller_id, customer_id)` de `identity.seller_customer`. Al crear la orden, una transacción bloqueada verifica que esa relación está `ACTIVE`; una inactivación posterior no invalida el histórico. No se crea la relación automáticamente. El XOR de beneficiario y la regla de que el Seller beneficiario es el propio comprador se protegen con `CHECK`.

#### `sales.order_item`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Línea comercial. |
| order_id | uuid | No | FK a `orders` | Orden propietaria. |
| product_id | uuid | No | FK compuesta con `product_offer_id` a `product_offer` | Producto snapshot/referencia. |
| product_name_snapshot | varchar(200) | No | Inmutable tras confirmación | Nombre vendido. |
| product_offer_id | uuid | No | FK compuesta con `product_id` a `product_offer` | Oferta vendida. |
| unit_price | numeric(19,4) | No | CHECK `unit_price >= 0`; inmutable tras confirmación | Precio unitario. |
| currency_code | char(3) | No | Igual a moneda de Order, validada transaccionalmente; inmutable | Moneda del ítem. |
| service_duration_days | integer | No | CHECK `service_duration_days > 0` | Duración contratada en días completos. |
| sale_mode_snapshot | varchar(20) | No | CHECK `PROFILE`/`FULL_ACCOUNT`; inmutable | Modalidad vendida. |
| quantity | integer | No | CHECK `quantity > 0`; y `previous_subscription_id IS NULL OR quantity = 1`; inmutable tras confirmar | Cantidad. |
| previous_subscription_id | uuid | Sí | FK a `sales.subscription`; nulo para compra regular | Período anterior objetivo de renovación/continuidad, incluso si Order sigue PENDING. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación permitida. |

La FK compuesta `(product_offer_id, product_id)` referencia la clave candidata `UNIQUE (id, product_id)` de `catalog.product_offer`; evita que producto y oferta pertenezcan a catálogos distintos sin duplicar IDs. La FK no compara `sale_mode_snapshot` con `product_offer.sale_mode`: un trigger `BEFORE INSERT OR UPDATE` de OrderItem bloquea primero su Order y luego la ProductOffer con `FOR SHARE`, exige igualdad exacta y rechaza la línea ante cualquier diferencia. El mismo orden evita carreras y deadlocks con la confirmación. Un trigger `BEFORE UPDATE` de `product_offer.sale_mode` rechaza la modificación desde la primera referencia en cualquier OrderItem, sin importar el estado de Order. Por tanto, cualquier cambio de modalidad posterior exige inactivar la oferta y crear una ProductOffer nueva; los snapshots e historial existente no se modifican.

#### `sales.subscription`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Período de servicio. |
| order_item_id | uuid | No | FK a `order_item` | Ítem originador. |
| start_date | date | No | — | Inicio. |
| expiration_date | date | No | CHECK `expiration_date > start_date` | Vencimiento. |
| previous_subscription_id | uuid | Sí | FK autorreferente; UNIQUE parcial; distinto de `id` | Período renovado, si aplica. |
| created_at | timestamptz | No | — | Creación. |

La transacción calcula duración desde `order_item.service_duration_days`, con `expiration_date = start_date + service_duration_days`. Inicio inclusivo, vencimiento exclusivo. La Subscription se crea al confirmar y `start_date` coincide con `businessDate`; una fecha futura no se admite.

La vigencia se deriva, sin `status_code`: `ACTIVE` cuando `start_date <= businessDate < expiration_date`, `EXPIRED` cuando `businessDate >= expiration_date`. `store.business_timezone` determina `businessDate`; no hay proceso que cambie el estado. Una línea de renovación/continuidad con `previous_subscription_id` solo confirma cuando `businessDate >= expiration_date` anterior; la Subscription copia la referencia y exige el mismo beneficiario/Product. `start_date = max(expiration_date anterior, businessDate)`. Un índice UNIQUE parcial en `sales.subscription.previous_subscription_id` permite como máximo un sucesor confirmado. El link y fechas son inmutables; el predecessor ya existe al insertar y start_date no precede su vencimiento, por lo que no hay ciclos.

`order_item.previous_subscription_id` registra la intención mientras la Order está `PENDING`; `subscription.previous_subscription_id` registra el servicio confirmado. Un trigger de confirmación verifica que ambos coincidan. El diseño no crea Subscription futura ni reserva inventario en una Order pendiente.

Toda Subscription persistida debe llegar al `COMMIT` con una y solo una asignación inicial válida: una fila de ResourceAssignmentHistory con `reason_code = INITIAL_ASSIGNMENT` y `sequence = 1`. Puede tener después cero o más `WARRANTY_REPLACEMENT`; estos nunca satisfacen el mínimo inicial. Constraint triggers `AFTER`, `DEFERRABLE INITIALLY DEFERRED`, en Subscription y ResourceAssignmentHistory lo garantizan al INSERT; el de historial también cubre borrado/cambio durante mantenimiento autorizado. Subscription no recibe estado ni actualización de expiración. El rol de aplicación no puede eliminarla; los registros históricos conservan Order/Item y RAH.
Una Order `PENDING` puede crearse sin ítems mientras se construye. Antes de confirmar, la transacción verifica que existe al menos un OrderItem. Cada línea confirmada origina exactamente `quantity` Subscriptions y asignaciones iniciales. Saldo insuficiente, inventario insuficiente, precio no disponible o cualquier fallo revierte toda la transacción: no quedan débito ni asignaciones parciales y la Order sigue `PENDING`. Tras confirmación no se modifica la composición. Solo `PENDING` puede cancelarse; para cancelaciones de servicio y compensaciones financieras se conservan Order y ledger, respectivamente.

### wallet

#### `wallet.wallet`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Billetera. |
| seller_id | uuid | No | FK a `seller`; UNIQUE | Seller titular. |
| balance | numeric(19,4) | No | CHECK `balance >= 0` | Saldo materializado. |
| currency_code | char(3) | No | CHECK de formato ISO | Moneda de la billetera. |
| next_sequence_no | bigint | No | CHECK `next_sequence_no > 0`; solo ledger | Próximo número contable de la Wallet. |
| created_at | timestamptz | No | — | Creación. |
| updated_at | timestamptz | No | — | Última modificación por movimiento. |

#### `wallet.wallet_transaction`

| Columna | Tipo | NULL permitido | Restricciones | Descripción |
| --- | --- | --- | --- | --- |
| id | uuid | No | PK | Movimiento inmutable. |
| wallet_id | uuid | No | FK a `wallet` | Billetera afectada. |
| sequence_no | bigint | No | UNIQUE con `wallet_id`; CHECK `sequence_no > 0`; asignado por ledger | Orden contable monotónico por Wallet. |
| order_id | uuid | Sí | FK a `sales.orders`; obligatorio solo para `PURCHASE_DEBIT`; UNIQUE parcial para ese tipo | Orden debitada. |
| reversed_wallet_transaction_id | uuid | Sí | FK a esta misma tabla; obligatorio solo para `REVERSAL`; único parcial | Movimiento original revertido. |
| type_code | varchar(30) | No | CHECK de tipos WalletTransaction | Tipo del movimiento. |
| amount | numeric(19,4) | No | CHECK de signo por tipo; distinto de cero | Delta firmado. |
| previous_balance | numeric(19,4) | No | CHECK `>= 0` | Saldo previo. |
| resulting_balance | numeric(19,4) | No | CHECK `>= 0` y ecuación de saldo | Saldo posterior. |
| currency_code | char(3) | No | CHECK de formato ISO | Moneda del movimiento. |
| reference | varchar(200) | Sí | — | Referencia administrativa o externa. |
| originated_by_user_id | uuid | No | FK a `identity.app_user` | Actor responsable. |
| occurred_at | timestamptz | No | — | Momento contable. |

`PURCHASE_DEBIT` exige `order_id`; `TOP_UP`, ambos `ADJUSTMENT_*` y `REVERSAL` exigen `order_id` nulo. Un índice único parcial sobre `order_id` para `PURCHASE_DEBIT` impide un segundo débito de compra para una misma Order, incluso ante reintentos concurrentes. Para un reverso, la orden se obtiene mediante el movimiento original. `reversed_wallet_transaction_id` es nulo para todo tipo diferente de `REVERSAL`; un índice único parcial sobre esa columna impide más de un reverso total por original.

`sequence_no` es el orden contable oficial, no `occurred_at`. El trigger de ledger bloquea Wallet, toma `next_sequence_no`, la asigna al nuevo movimiento y aumenta el contador dentro de la misma transacción que valida/importa saldo y persiste el movimiento. Por ello todos los tipos —TOP_UP, PURCHASE_DEBIT, ADJUSTMENT_CREDIT, ADJUSTMENT_DEBIT y REVERSAL— reciben una secuencia estrictamente creciente por Wallet. La unicidad `UNIQUE (wallet_id, sequence_no)` es defensa declarativa; la asignación no usa una secuencia global de PostgreSQL. Un rollback deshace movimiento, contador y saldo, de modo que no deja un orden contable confirmado artificial.

WalletTransaction no tiene `status_code`. Para CU-008, una consulta presenta `REVERSAL` para la fila compensatoria; para el movimiento original deriva `REVERSED` cuando existe un `REVERSAL` cuyo `reversed_wallet_transaction_id` apunta a él y `NOT_REVERSED` en otro caso.

Solo pueden revertirse `TOP_UP`, `PURCHASE_DEBIT`, `ADJUSTMENT_CREDIT` y `ADJUSTMENT_DEBIT`; un `REVERSAL` nunca es reversible. En la transacción bloqueada se exige mismo `wallet_id`, moneda igual, importe exactamente opuesto, original no revertido y balance resultante válido. Si después del reverso se requiere corregir otro hecho, se registra un nuevo `ADJUSTMENT_CREDIT` o `ADJUSTMENT_DEBIT` autorizado: jamás se modifica un movimiento histórico.

No se adopta una FK compuesta Wallet–Order porque, con las columnas cerradas, requeriría duplicar el Seller comprador en WalletTransaction. La garantía principal es transaccional: se bloquean la Order `PENDING` y la Wallet candidata, se verifica `wallet.seller_id = orders.buyer_seller_id`, moneda, saldo e inexistencia de `PURCHASE_DEBIT` previo; después se inserta el movimiento y se actualiza Wallet de forma atómica. La unicidad parcial convierte un reintento concurrente en una operación rechazada sin segundo descuento.

Todo Seller persistido debe tener exactamente una Wallet. `UNIQUE (wallet.seller_id)` protege el máximo y la FK protege la existencia del Seller, pero ninguna de las dos protege el mínimo. La alta crea `User + rol SELLER + Seller + Wallet` en una sola transacción; Wallet inicia con `balance = 0`, `next_sequence_no = 1` y no se crea un `TOP_UP` ficticio. Constraint triggers `AFTER` diferidos, `DEFERRABLE INITIALLY DEFERRED`, son la garantía de mínimo al `COMMIT`: el de Seller verifica que la nueva fila tiene exactamente una Wallet y el de Wallet revalida al Seller afectado en INSERT, UPDATE de `seller_id` o DELETE. El trigger de Wallet rechaza además cambiar `seller_id` o eliminarla mientras exista su Seller; DELETE de Seller está prohibido por permisos y por sus referencias históricas. Así no se introduce una FK cíclica ni un estado confirmado con Seller sin Wallet; si falla cualquier paso del alta, la transacción completa hace rollback.

## Integridad transaccional, inmutabilidad y permisos

Se usa `READ COMMITTED`, transacciones PostgreSQL, bloqueo pesimista de filas candidatas, revalidación y rollback completo. Una compra bloquea recurso y Wallet, revalida actores, catálogo, precio, inventario, moneda y saldo; después persiste Order, ítems, Subscription, historial y WalletTransaction de forma atómica. Las escrituras de Account, Profile y ResourceAssignmentHistory usan obligatoriamente la frontera de inventario y el protocolo de locks que sigue; no son DML libre de la aplicación. Los reemplazos bloquean los recursos anterior/nuevo, Subscription e historial conforme a ese orden y crean el reemplazo en la misma transacción.

Las restricciones entre filas o tablas no se sustituyen por un `CHECK`: moneda Wallet/Order/OrderItem/WalletTransaction, estado `ACTIVE` de SellerCustomer al crear, elegibilidad de credenciales, razón y secuencia de ResourceAssignmentHistory, compatibilidad de Platform y la exclusión FULL_ACCOUNT/Profile se validan bajo esos bloqueos.

La defensa de inmutabilidad es en capas:

* La aplicación no actualiza ni elimina `wallet.wallet_transaction`, Order confirmada, snapshots de Order/OrderItem confirmada ni filas históricas de ResourceAssignmentHistory.
* El rol de aplicación recibe privilegios mínimos: no `DELETE` sobre estas tablas y no `UPDATE` sobre WalletTransaction ni sobre campos protegidos de Order/OrderItem/ResourceAssignmentHistory. En `inventory.account`, `inventory.profile` e `inventory.resource_assignment_history` tampoco recibe `INSERT`/`UPDATE`/`DELETE` directo: solo `EXECUTE` de las funciones de frontera autorizadas. El propietario de esquema se reserva para mantenimiento controlado.
* Triggers obligatorios rechazan `UPDATE`/`DELETE` de WalletTransaction; protegen una Order confirmada y sus ítems/snapshots; y hacen ResourceAssignmentHistory append-only, salvo completar una sola vez `released_at`, `released_by_user_id` y `release_reason_code`. Order solo permite `PENDING → CONFIRMED` o `PENDING → CANCELLED`; Subscription no tiene transición persistida. `wallet.balance` solo cambia al insertar el movimiento correspondiente.

Los índices operativos mínimos cubren FKs, búsquedas por estado/vencimiento, WalletTransaction por `(wallet_id, occurred_at)`, y los recursos disponibles. No hay cascadas físicas para datos comerciales, financieros ni históricos.

## Secuencias transaccionales y garantías por capa

### Confirmación comercial con Wallet

El orden obligatorio es: (1) bloquear la Order `PENDING`; (2) bloquear sus OrderItem y comprobar que hay al menos uno; (3) bloquear sus ProductOffer con `FOR SHARE`, validar `sale_mode_snapshot = sale_mode`, y bloquear PriceList y PriceListItem aplicables para recalcular el precio al instante de confirmación; (4) bloquear Account raíz y Profile candidato, según modalidad, y revalidar inventario, Platform y correspondencia de modo; (5) bloquear la Wallet del buyer; (6) verificar propietario, saldo y moneda; (7) comprobar que no existe `PURCHASE_DEBIT` para la Order; (8) insertar el movimiento con `previous_balance` igual al saldo bloqueado y `resulting_balance = previous_balance + amount`; (9) el trigger de ledger actualiza Wallet a ese resultado y se crean/confirman Order, Subscription e historial; y (10) confirmar. Cualquier error, incluido conflicto de unicidad, revierte todo. Los triggers de validación solo leen y bloquean contexto, o actualizan Wallet desde el flujo de ledger; no se invocan mutuamente ni reinsertan ResourceAssignmentHistory, por lo que no hay recursión.

`Wallet.balance` es el saldo materializado actual; si no hay movimientos vale cero. `previous_balance` es el saldo inmediatamente anterior a su movimiento, `amount` es el delta firmado y `resulting_balance` es el saldo inmediatamente posterior. El movimiento de mayor `sequence_no` es el último contable y su `resulting_balance` debe igualar `Wallet.balance`. Un CHECK protege ecuación, signos y no negatividad de la fila; no puede verificar otra tabla. La serialización se logra bloqueando la misma Wallet y revalidando saldo/último movimiento. Un trigger de ledger `BEFORE INSERT` en WalletTransaction bloquea Wallet, asigna `sequence_no = next_sequence_no`, comprueba que el saldo bloqueado coincide con `previous_balance`, incrementa el contador y actualiza Wallet al resultado dentro de esa misma transacción. El rol de aplicación no tiene UPDATE directo sobre `wallet.balance` ni `next_sequence_no`; un trigger de guardia en Wallet rechaza toda modificación que no proceda del flujo de ledger. Si falla cualquiera de estas comprobaciones o la inserción posterior, se revierte contador, saldo y movimiento.

En el esquema nuevo no existe backfill porque no hay movimientos históricos. Si en una implementación posterior hubiera datos preexistentes sin `sequence_no`, no se infiere orden desde `occurred_at`: la migración debe detenerse hasta disponer de una fuente externa y verificable de orden contable, o tratar esos datos mediante una migración de conciliación aprobada.

### Precio snapshot y autorización

El instante comercial de precio es la confirmación. Bajo los locks de PriceList/PriceListItem, la transacción verifica rol objetivo del buyer, lista `ACTIVE`, oferta correspondiente, vigencia `[valid_from, valid_to)`, prioridad máxima inequívoca, moneda de Order y coincidencia exacta entre importe/moneda calculados y el snapshot de OrderItem. Toda alta, cambio de vigencia/prioridad o cambio de estado de PriceList/PriceListItem bloquea primero su PriceList; así también se serializa la inserción concurrente de una prioridad diferente. La exclusión GiST impide empates de igual prioridad; el bloqueo y la revalidación impiden que una edición concurrente confirme un precio obsoleto o ambiguo. Cambios posteriores no alteran una Order confirmada.

La FK de `originated_by_user_id` prueba identidad, no autorización. Dentro del flujo transaccional se bloquea y valida User `ACTIVE` y `user_role`: `TOP_UP` y `ADJUSTMENT_*` requieren `ADMIN`; `PURCHASE_DEBIT` requiere actor autorizado para el Seller buyer; `REVERSAL` requiere actor autorizado para la operación revertida. La aplicación es la garantía principal de autorización contextual; triggers de WalletTransaction son defensa adicional para tipo/rol/estado. Un CUSTOMER no puede iniciar recargas ni ajustes administrativos.

### Estado de inventario

#### Protocolo global de locks y frontera de escritura

La frontera de escritura de inventario es obligatoria y reside en funciones PostgreSQL autorizadas; no son recomendaciones para Java. Cada función se ejecuta íntegramente en la transacción llamadora —no inicia ni confirma una transacción autónoma—, tiene propietario dedicado sin privilegios de login, usa `SECURITY DEFINER` solo cuando necesita escribir en las tablas protegidas, fija un `search_path` seguro a `pg_catalog` y usa nombres de esquema calificados. `PUBLIC` no recibe `EXECUTE`; el rol de aplicación recibe únicamente las funciones de negocio autorizadas, y carece de DML directo sobre Account, Profile y ResourceAssignmentHistory. El propietario de migración y el rol de mantenimiento controlado quedan fuera de la frontera y no son roles de aplicación.

Antes de cualquier DML hijo, las funciones toman `FOR UPDATE` sobre todas las Accounts raíz afectadas, ordenadas ascendentemente por UUID. Después toman `FOR UPDATE` sobre Profiles existentes afectados, ordenados por UUID; luego sobre filas existentes de ResourceAssignmentHistory, ordenadas por `(subscription_id, sequence)`; y finalmente sobre Subscriptions existentes afectadas, ordenadas por UUID. Solo después emiten el `INSERT`, `UPDATE` o `DELETE` objetivo. Las filas nuevas no tienen lock previo; sus padres y recursos sí. Los locks comerciales de Order/OrderItem/ProductOffer se toman antes de esta frontera durante confirmación, y Wallet después de la frontera, como ya indica la secuencia comercial. Ninguna función de inventario toma Account, Profile o historial en otro orden.

Las únicas entradas de escritura son: (1) provisionar Account con Profiles iniciales; (2) agregar, trasladar, inactivar o retirar un Profile sin historial; (3) crear Subscription con INITIAL_ASSIGNMENT; (4) reemplazar por garantía, liberando la fila vigente y creando WARRANTY_REPLACEMENT; (5) liberar una asignación con razón autorizada; y (6) cambiar estado/datos permitidos de Account/Profile. Subscription no tiene transiciones persistidas. La expiración solo cambia el resultado de la consulta por fechas; no modifica Subscription ni libera stock. Una operación administrativa posterior de liberación usa la misma frontera. El trigger de fila conserva validaciones defensivas, pero no adquiere Account tardíamente.

| Operación autorizada | Frontera y locks previos, en este orden | Filas modificadas y controles al COMMIT |
| --- | --- | --- |
| Provisionar Account con Profiles iniciales | Función de alta; bloquea Platform con `FOR SHARE`; crea la nueva Account y, antes de insertar hijos, mantiene su lock de escritura. No hay Account preexistente que ordenar. | `inventory.account`, uno o más `inventory.profile`. Constraint trigger Account/Profile exige el mínimo al COMMIT. |
| Agregar, trasladar, inactivar o retirar Profile sin historial | Función de Profile; Accounts origen/destino `FOR UPDATE` por UUID, Profile existente `FOR UPDATE`, historial afectado `FOR UPDATE`, Subscription afectada `FOR UPDATE`. | `inventory.profile`; los triggers protegen plataforma, historial y estado. El constraint trigger revalida ambas Accounts al COMMIT. |
| Crear Subscription con INITIAL_ASSIGNMENT | Función de confirmación/asignación; mantiene primero locks comerciales existentes, luego Account raíz `FOR UPDATE`, Profile candidato `FOR UPDATE` si aplica, historial conflictivo existente y finalmente Subscription existente si la operación no la crea. | `sales.subscription` nueva y `inventory.resource_assignment_history` inicial. Constraint trigger diferido exige INITIAL_ASSIGNMENT secuencia 1 al COMMIT. |
| Reposición por garantía | Función de garantía; todas las Accounts raíz (anterior y nueva) `FOR UPDATE` por UUID, Profiles, historial vigente y Subscription `FOR UPDATE`. | Se completan `released_at`, `released_by_user_id` y `release_reason_code = WARRANTY_REPLACEMENT` en la fila previa; se inserta `WARRANTY_REPLACEMENT`. Modo, Platform, secuencia y exclusiones se revalidan antes del DML y al COMMIT. |
| Liberar asignación autorizada | Función de liberación; Account raíz `FOR UPDATE`, Profile si aplica `FOR UPDATE`, fila de historial `FOR UPDATE`, Subscription `FOR UPDATE`. | Completa una vez `released_at`, actor y motivo (`RENEWAL_TRANSFER` o `MANUAL_RECONDITIONING`). No borra la INITIAL_ASSIGNMENT. |
| Cambio administrativo de Account o Profile | Función administrativa; Accounts raíz `FOR UPDATE` por UUID, Profiles afectados `FOR UPDATE`, historial vigente `FOR UPDATE`, Subscription afectada `FOR UPDATE` si existe. | Solo atributos/estados permitidos. Rechaza estados incompatibles; no permite mover Profile con historial. |
| Consultar estado o expiración de Subscription | Consulta por fechas con `businessDate`; no modifica Subscription. Si una renovación reutiliza un recurso, la función de inventario libera el RAH anterior y crea la nueva asignación inicial de forma atómica. | Estado derivado, sin job ni mutación al vencer; la liberación conserva actor y motivo. |

Esto evita la inversión observada: PostgreSQL puede bloquear la fila objetivo antes del trigger `BEFORE`, pero la función ya obtuvo los locks de Account/Profile/historial en orden canónico antes de emitir ese DML. Una actualización administrativa de Account no puede retener Account mientras espera un Profile que otra transacción modificó sin haber bloqueado primero Account, porque ese DML directo está revocado. Si una función encuentra un deadlock o error de serialización, la transacción completa se revierte; el llamador puede reintentar la operación completa, nunca un paso parcial.

Toda transición de Account o Profile revalida asignaciones vigentes y la exclusión FULL_ACCOUNT/Profile antes de persistir. No se inactiva un recurso con asignación vigente si contradice el servicio; primero se libera mediante el proceso de negocio correspondiente. El stock continúa siendo derivado de estados y asignaciones vigentes; no existe contador manual.

| Invariante | Garantía principal | Defensa adicional |
| --- | --- | --- |
| Existencia de referencias | PK/FK `RESTRICT` | Índices de FK. |
| Unicidad de débito y reverso | UNIQUE parcial | Lock de Order/Wallet y revalidación. |
| Ecuación y signo de movimiento | CHECK | Trigger de balance y transacción. |
| Propietario Wallet = buyer | Transacción y lock | Validación de aplicación. |
| Rol de Seller/Customer | Transacción | Triggers en perfiles y user_role. |
| Mínimo de OrderItem al confirmar | Transacción | Trigger de transición de Order. |
| Mínimo e identidad de Profiles | Constraint trigger diferido y lock de Account | Triggers/privilegios. |
| Asignación inicial obligatoria de Subscription | Constraint triggers diferidos de Subscription e historial | Frontera de escritura y validación de `INITIAL_ASSIGNMENT`, secuencia y modo. |
| Precio temporal no ambiguo | EXCLUDE/GiST | Lock y revalidación al confirmar. |
| Inventario simultáneo | Índices parciales/EXCLUDE | Lock pesimista y revalidación. |
| Orden contable por Wallet | Trigger BEFORE INSERT y lock de Wallet | UNIQUE `(wallet_id, sequence_no)` y permisos. |
| Historial inmutable | Permisos | Triggers de UPDATE/DELETE. |

### Catálogo de triggers obligatorios

| Tabla | Evento | Invariante y relación con la transacción |
| --- | --- | --- |
| `identity.seller` / `identity.customer` | BEFORE INSERT, BEFORE UPDATE de `user_id` | Exige el rol correspondiente en `user_role`; complementa el alta atómica. |
| `identity.user_role` | BEFORE DELETE o UPDATE de `role_code` | Impide retirar `SELLER`/`CUSTOMER` si existe el perfil dependiente. |
| `wallet.wallet` y `identity.seller` | Constraint trigger AFTER, DEFERRABLE INITIALLY DEFERRED | Al COMMIT verifica exactamente una Wallet por Seller; cubre alta, cambio o intento de eliminación sin FK cíclica. |
| `wallet.wallet` | BEFORE UPDATE de `seller_id`, BEFORE DELETE | Impide mover o eliminar la Wallet obligatoria mientras exista el Seller. |
| `inventory.account` / `inventory.profile` | Constraint trigger AFTER, DEFERRABLE INITIALLY DEFERRED | Al COMMIT exige al menos un Profile asociado por Account; Account INSERT y Profile INSERT/DELETE/traslado revalidan las Accounts afectadas. |
| `inventory.account` | BEFORE UPDATE de `platform_id` o `is_active`; BEFORE DELETE | Rechaza cambiar Platform después de historial, inactivar con asignación vigente o una baja que eluda Profiles/historial. |
| `inventory.profile` | BEFORE INSERT, DELETE; BEFORE UPDATE de `account_id` o `is_active` | Protege mínimo, inmutabilidad tras historial e inactivación incompatible; no adquiere locks raíz tardíos. |
| `catalog.product_offer` | BEFORE UPDATE de `sale_mode` | Impide cambiar el modo desde la primera referencia de cualquier OrderItem. |
| `sales.orders` | BEFORE UPDATE de `status_code` | Limita transiciones y exige al menos un OrderItem al confirmar. |
| `sales.order_item` | BEFORE INSERT o UPDATE de campos comerciales; BEFORE DELETE | Bloquea Order y ProductOffer, valida modo snapshot y prohíbe modificar composición de una Order CONFIRMED. |
| `sales.subscription` | BEFORE INSERT; BEFORE UPDATE de `previous_subscription_id`, `order_item_id` o fechas | Valida fecha de inicio en `businessDate`, copia del predecessor desde OrderItem, beneficiary/Product y unicidad del sucesor; las fechas y vínculos son inmutables tras insertar. El estado es una vista/expresión calculada. |
| `sales.subscription` / `inventory.resource_assignment_history` | Constraint trigger AFTER, DEFERRABLE INITIALLY DEFERRED | Al COMMIT exige una INITIAL_ASSIGNMENT válida (`sequence = 1`) por Subscription. |
| `inventory.resource_assignment_history` | BEFORE INSERT; BEFORE UPDATE o DELETE | Valida modo, recurso, Platform y secuencia; es append-only salvo completar una única vez `released_at`, `released_by_user_id` y `release_reason_code` de una fila retenida. |
| `identity.audit_event` | BEFORE UPDATE o DELETE | Rechaza mutación o borrado de eventos por la aplicación. |
| `wallet.wallet_transaction` | BEFORE INSERT; BEFORE UPDATE/DELETE | Mantiene movimientos inmutables; asigna secuencia, verifica saldo previo, contador y balance dentro de la misma transacción. |
| `wallet.wallet` | BEFORE UPDATE de `balance` o `next_sequence_no` | Rechaza cambios que no procedan del flujo de ledger; el rol de aplicación carece además de UPDATE directo. |

## Inventario de índices y restricciones indexadas

Los nombres son lógicos para la futura migración; no son SQL ejecutable. El inventario contiene **66 estructuras indexadas**: 19 índices de PK, 18 índices `UNIQUE` adicionales, 3 restricciones `EXCLUDE` y 26 índices explícitos no derivados de restricción. PK y UNIQUE crean su índice y no se duplican por igual prefijo. `CHECK`, FK y triggers no crean índices por sí mismos.

### Índices de PRIMARY KEY (19)

`pk_app_user`, `pk_user_role`, `pk_audit_event`, `pk_seller`, `pk_customer`, `pk_seller_customer`, `pk_platform`, `pk_product`, `pk_product_offer`, `pk_price_list`, `pk_price_list_item`, `pk_account`, `pk_profile`, `pk_resource_assignment_history`, `pk_orders`, `pk_order_item`, `pk_subscription`, `pk_wallet` y `pk_wallet_transaction`. Todos son btree sobre la PK declarada.

### Índices UNIQUE adicionales (18)

`uq_app_user_email_ci`, `uq_seller_user`, `uq_customer_user`, `uq_seller_customer_pair`, `uq_product_offer_id_product`, `uq_product_offer_active_mode`, `uq_price_list_active_role`, `uq_profile_account_identifier`, `uq_rah_subscription_sequence`, `uq_rah_current_subscription`, `uq_rah_current_profile`, `uq_rah_current_full_account`, `uq_orders_order_number`, `uq_subscription_renewal_parent`, `uq_wallet_seller`, `uq_wallet_tx_purchase_order`, `uq_wallet_tx_reversal_original` y `uq_wallet_tx_sequence`.

### Restricciones EXCLUDE con índice (3)

`ex_price_list_item_no_overlap`, `ex_rah_profile_time` y `ex_rah_full_account_time` se materializan mediante GiST. Las exclusiones de ResourceAssignmentHistory son restricciones e índices independientes.

### Índices explícitos no derivados de restricción (26)

Son los índices con prefijo `ix_` del catálogo siguiente. El total de estructuras es 19 PK + 18 UNIQUE + 3 EXCLUDE + 26 explícitas = 66.

| Esquema.Tabla | Nombre lógico del índice | Columnas/expresión | Tipo | Parcial o completo | Propósito |
| --- | --- | --- | --- | --- |
| identity.app_user | uq_app_user_email_ci | email normalizado | UNIQUE btree | Completo | Login sin distinción de mayúsculas. |
| identity.audit_event | ix_audit_event_actor_time | actor_user_id, occurred_at | btree | Completo | Auditoría por actor y período. |
| identity.audit_event | ix_audit_event_entity_time | entity_type, entity_id, occurred_at | btree | Completo | Historial de un objeto. |
| identity.audit_event | ix_audit_event_occurred | occurred_at | btree | Completo | Consulta y retención por tiempo. |
| identity.seller | uq_seller_user | user_id | UNIQUE btree | Completo | Perfil Seller único por User. |
| identity.customer | uq_customer_user | user_id | UNIQUE btree | Completo | Perfil Customer único por User. |
| identity.seller_customer | uq_seller_customer_pair | seller_id, customer_id | UNIQUE btree | Completo | Relación comercial y FK compuesta de Order. |
| identity.seller_customer | ix_seller_customer_customer | customer_id | btree | Completo | Relaciones de un Customer. |
| catalog.platform | ix_platform_status_name | status_code, name | btree | Completo | Catálogo operativo. |
| catalog.product | ix_product_platform_status | platform_id, status_code | btree | Completo | Productos elegibles por Platform. |
| catalog.product_offer | uq_product_offer_id_product | id, product_id | UNIQUE btree | Completo | Destino de FK compuesta de OrderItem. |
| catalog.product_offer | uq_product_offer_active_mode | product_id, sale_mode | UNIQUE btree | Parcial `ACTIVE` | Una oferta activa por modalidad. |
| catalog.product_offer | ix_product_offer_product_status | product_id, status_code | btree | Completo | Ofertas elegibles del Product. |
| catalog.price_list | uq_price_list_active_role | target_role | UNIQUE btree | Parcial `ACTIVE` | Una lista activa por rol. |
| catalog.price_list | ix_price_list_role_status | target_role, status_code | btree | Completo | Resolución de lista aplicable. |
| catalog.price_list_item | ex_price_list_item_no_overlap | price_list_id, product_offer_id, priority, rango de vigencia | EXCLUDE GiST | Completo | Impide solapamiento de igual prioridad. |
| catalog.price_list_item | ix_price_list_item_lookup | price_list_id, product_offer_id, valid_from | btree | Completo | Selección y revalidación de precio. |
| inventory.account | ix_account_platform_active | platform_id, is_active | btree | Completo | Cuentas habilitadas por Platform. |
| inventory.account | ix_account_enabled | platform_id, id | btree | Parcial `is_active = true` | Candidatas base a FULL_ACCOUNT; la vista valida disponibilidad. |
| inventory.profile | uq_profile_account_identifier | account_id, profile_identifier | UNIQUE btree | Completo | Identidad del Profile. |
| inventory.profile | ix_profile_account_active | account_id, is_active | btree | Completo | Perfiles habilitados de una Account. |
| inventory.profile | ix_profile_enabled | account_id, id | btree | Parcial `is_active = true` | Candidatas base PROFILE; la vista valida disponibilidad. |
| inventory.resource_assignment_history | uq_rah_subscription_sequence | subscription_id, sequence | UNIQUE btree | Completo | Secuencia histórica. |
| inventory.resource_assignment_history | uq_rah_current_subscription | subscription_id | UNIQUE btree | Parcial `released_at IS NULL` | Un recurso vigente por Subscription. |
| inventory.resource_assignment_history | uq_rah_current_profile | profile_id | UNIQUE btree | Parcial `released_at IS NULL` y profile presente | Un uso vigente por Profile. |
| inventory.resource_assignment_history | uq_rah_current_full_account | account_id | UNIQUE btree | Parcial `released_at IS NULL` y account presente | Un FULL_ACCOUNT vigente. |
| inventory.resource_assignment_history | ex_rah_profile_time | profile_id, rango temporal | EXCLUDE GiST | Parcial profile presente | Sin solapes del mismo Profile. |
| inventory.resource_assignment_history | ex_rah_full_account_time | account_id, rango temporal | EXCLUDE GiST | Parcial account presente | Sin solapes FULL_ACCOUNT. |
| inventory.resource_assignment_history | ix_rah_account_current | account_id, released_at | btree | Parcial account presente | Revalidación FULL_ACCOUNT/Profile. |
| inventory.resource_assignment_history | ix_rah_actor_assigned_at | performed_by_user_id, assigned_at | btree | Completo | Auditoría operativa por actor. |
| sales.orders | uq_orders_order_number | order_number | UNIQUE btree | Completo | Identificador comercial. |
| sales.orders | ix_orders_buyer_status_created | buyer_seller_id, status_code, created_at | btree | Completo | Compras del Seller y confirmación. |
| sales.orders | ix_orders_beneficiary_seller | beneficiary_seller_id | btree | Parcial beneficiario Seller | Consulta histórica por beneficiario. |
| sales.orders | ix_orders_beneficiary_customer | beneficiary_customer_id | btree | Parcial beneficiario Customer | Consulta histórica por beneficiario. |
| sales.order_item | ix_order_item_order | order_id | btree | Completo | Bloqueo y lectura de ítems de Order. |
| sales.order_item | ix_order_item_offer | product_offer_id | btree | Completo | Historial por oferta. |
| sales.order_item | ix_order_item_previous_subscription | previous_subscription_id | btree | Parcial valor presente | Consultar intentos de renovación/continuidad por período anterior. |
| sales.subscription | ix_subscription_order_item | order_item_id | btree | Completo | Períodos originados por ítem. |
| sales.subscription | ix_subscription_expiration_date | expiration_date | btree | Completo | Búsquedas de expiración por rango; estado se deriva. |
| sales.subscription | uq_subscription_renewal_parent | previous_subscription_id | UNIQUE btree | Parcial valor presente | Máximo un sucesor de renovación. |
| wallet.wallet | uq_wallet_seller | seller_id | UNIQUE btree | Completo | Una Wallet por Seller. |
| wallet.wallet_transaction | uq_wallet_tx_purchase_order | order_id | UNIQUE btree | Parcial `type_code = PURCHASE_DEBIT` | Idempotencia del débito. |
| wallet.wallet_transaction | uq_wallet_tx_reversal_original | reversed_wallet_transaction_id | UNIQUE btree | Parcial valor presente | Un reverso total por original. |
| wallet.wallet_transaction | uq_wallet_tx_sequence | wallet_id, sequence_no | UNIQUE btree | Completo | Orden contable único y reconstrucción determinista del ledger. |
| wallet.wallet_transaction | ix_wallet_tx_wallet_occurred | wallet_id, occurred_at | btree | Completo | Consultas cronológicas de auditoría; no define el orden contable. |
| wallet.wallet_transaction | ix_wallet_tx_order | order_id | btree | Parcial valor presente | Auditoría de débito por Order. |
| wallet.wallet_transaction | ix_wallet_tx_actor_occurred | originated_by_user_id, occurred_at | btree | Completo | Auditoría por actor. |

## Alcance

El stock es derivado: Profiles `AVAILABLE` para PROFILE y Accounts `AVAILABLE` para FULL_ACCOUNT, con las validaciones anteriores. Vencer una Subscription no devuelve stock automáticamente. Una renovación crea nueva Order, OrderItem y Subscription; una garantía agrega historial a la misma Subscription. No existen tablas `Admin`, `Stock`, `Payment`, `Renewal`, `Guarantee`, `Replacement` ni `Reconditioning` en el MVP.
