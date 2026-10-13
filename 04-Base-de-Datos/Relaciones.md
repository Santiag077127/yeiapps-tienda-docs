# Relaciones — Base conceptual

Este documento define las relaciones y cardinalidades conceptuales del dominio. No prescribe tablas, claves, tipos, restricciones SQL ni estrategias de persistencia.

## Cardinalidades conceptuales definitivas

| Relación | Cardinalidad | Naturaleza | Observaciones |
| --- | --- | --- | --- |
| [[User]] → [[Seller]] | 1 : 0..1 | Principal | Cada Seller pertenece a un único User; solo existe para un User con rol `SELLER`. |
| [[User]] → [[Customer]] | 1 : 0..1 | Principal | Cada Customer pertenece a un único User; solo existe para un User con rol `CUSTOMER`. |
| [[User]] → `UserRole` | 1 : 0..N | Identidad | Roles asignados de un catálogo cerrado (`ADMIN`, `SELLER`, `CUSTOMER`), sin entidad Role configurable. |
| [[User]] → `AuditEvent` como actor | 1 : 0..N | Auditoría | Eventos realizados por este User; los eventos de sistema tienen actor nulo. |
| `AuditEvent.actorUserId` → [[User]] | 0..1 User por evento | Auditoría | Es obligatorio si `actorType = USER` y nulo si `actorType = SYSTEM`; un User puede originar 0..N eventos. |
| [[Seller]] → `SellerCustomer` | 1 : 0..N | Relación | Un Seller puede no tener Customers o relacionarse con varios. |
| [[Customer]] → `SellerCustomer` | 1 : 0..N | Relación | Un Customer puede no tener Sellers o relacionarse con varios. `SellerCustomer` expresa la N:N; Customer no tiene un `sellerId` único. |
| [[Seller]] → [[Wallet]] | 1 : 1 | Principal | Solo Seller tiene Wallet; Customer no tiene saldo interno. |
| [[Wallet]] → [[WalletTransaction]] | 1 : 0..N | Histórica | Los movimientos son inmutables y no se eliminan físicamente. |
| [[Platform]] → [[Account]] | 1 : 0..N | Operativa | Cada Account pertenece a una única Platform. |
| [[Platform]] → [[Product]] | 1 : 0..N | Catálogo | Cada Product pertenece a exactamente una Platform. |
| [[Account]] → [[Profile]] | 1 : 1..N | Operativa | Cada Profile pertenece a una única Account. |
| [[Product]] → [[ProductOffer]] | 1 : 0..N | Catálogo | Una oferta pertenece a un único Product; un producto puede estar aún sin oferta comercial publicada. |
| [[PriceList]] → [[PriceListItem]] | 1 : 0..N | Catálogo | Cada ítem pertenece a una única lista; una lista puede existir sin ítems vigentes. |
| [[ProductOffer]] → [[PriceListItem]] | 1 : 0..N | Catálogo | Cada ítem cotiza una única oferta; una oferta puede no tener precio vigente en una lista dada. |
| [[Seller]] → [[Order]] como buyer | 1 : 0..N | Histórica comercial | Cada Order del MVP tiene exactamente un Seller comprador. |
| [[Seller]] / [[Customer]] → [[Order]] como beneficiary | 1 : 0..N, excluyente | Histórica comercial | Cada Order tiene exactamente un beneficiario: el Seller comprador o un Customer con relación `SellerCustomer` válida. |
| [[Order]] → [[OrderItem]] | 1 : 0..N | Histórica comercial | Una Order `PENDING` puede tener cero ítems mientras se construye; toda Order confirmada debe tener uno o más. Cada ítem pertenece a una única Order. |
| [[OrderItem]] → [[Subscription]] | 1 : 0..N | Histórica comercial | En Order PENDING hay cero períodos; al confirmar se originan exactamente `quantity` períodos, cada uno con una asignación inicial. |
| [[Subscription]] → [[OrderItem]] que la continúa | 1 : 0..N | Intención comercial | Una línea `PENDING` puede conservar el período objetivo; se permite cancelar y reintentar. Una línea continuada exige `quantity = 1`. |
| [[Subscription]] → Subscription previa | 0..1 : 0..1 | Linaje confirmado | La Subscription nueva copia el vínculo del OrderItem; un período admite como máximo un sucesor confirmado. |
| [[Subscription]] → [[ResourceAssignmentHistory]] | 1 : 1..N | Asociación histórica | Exactamente una `INITIAL_ASSIGNMENT` de secuencia 1 y cero o más reemplazos dentro del período; se garantiza al COMMIT. |
| [[ResourceAssignmentHistory]] → [[Account]] / [[Profile]] | 1 : 1 excluyente | Asociación histórica | Cada asignación usa exactamente una Account para `FULL_ACCOUNT` o exactamente un Profile para `PROFILE`, nunca ambos. La liberación conserva fecha, actor y motivo. |
| [[Account]] / [[Profile]] → [[ResourceAssignmentHistory]] | 1 : 0..N | Asociación histórica | Un recurso puede participar en asignaciones históricas a lo largo del tiempo. |
| [[Profile]] → asignación vigente | 1 : 0..1 | Derivada operativa | Un Profile no puede tener dos asignaciones vigentes. |
| [[Account]] → asignación `FULL_ACCOUNT` vigente | 1 : 0..1 | Derivada operativa | Una Account no puede tener dos asignaciones completas vigentes. |
| [[Account]] → asignaciones `PROFILE` vigentes | 1 : 0..N | Derivada operativa | Puede tener varias cuando cada una usa un Profile diferente. |
| [[Subscription]] → recurso actual | 1 : 1 derivada | Derivada | Es la fila más reciente de ResourceAssignmentHistory sin fecha de liberación; permanece retenida hasta transferencia o publicación manual, incluso si el servicio expiró. |

## Reglas de lectura obligatorias

* `ADMIN` es `User + Role(ADMIN)`; no existe entidad `Admin`. Los únicos roles son `ADMIN`, `SELLER` y `CUSTOMER`.
* `SellerCustomer` es una relación conceptual explícita, no un Customer con `sellerId` exclusivo.
* Al crear una Order cuyo beneficiary es Customer, la relación `(Seller, Customer)` debe estar `ACTIVE` y se valida transaccionalmente; la inactivación posterior no invalida la operación histórica ni se crea la relación automáticamente.
* Cuando el beneficiary es Seller, la referencia debe existir y corresponder al buyer. Cuando es Customer, la FK compuesta conserva su relación con el buyer. Las referencias históricas no se eliminan.
* `Order` conserva solo el contexto común de la operación. Los snapshots de producto, oferta, precio, duración, modalidad y cantidad pertenecen a [[OrderItem]].
* El recurso operativo no se asigna desde OrderItem: se registra en [[ResourceAssignmentHistory]].
* `PROFILE` usa un Profile; `FULL_ACCOUNT` usa una Account. Cada registro de ResourceAssignmentHistory referencia exactamente uno de ambos, nunca los dos.
* Una asignación `FULL_ACCOUNT` vigente excluye cualquier asignación vigente de los Profiles de esa Account. Esta exclusión requiere bloqueo, consulta de asignaciones vigentes, revalidación y persistencia en la misma transacción; no se resuelve solo con un índice UNIQUE simple.
* Una renovación/continuidad crea nueva `Order`, `OrderItem` y `Subscription`; OrderItem conserva `previousSubscriptionId` mientras está pendiente y Subscription lo copia al confirmarse. Beneficiario y Product deben coincidir. Solo se confirma cuando `businessDate >= expirationDate` previa y `startDate = max(expirationDate anterior, businessDate)`. No hay pago, asignación ni período futuro antes de confirmar.
* Una garantía agrega un reemplazo al historial de la misma Subscription. La fila anterior identifica el recurso fallido; la nueva, el recurso sustituto. No crea Order, OrderItem ni Subscription.
* [[Product]] configura la renovabilidad, pero la trazabilidad comercial hacia Subscription es `Product → ProductOffer → OrderItem → Subscription`; no hay relación directa Product → Subscription en el ER.
* Cada [[Product]] pertenece a una [[Platform]]. La elegibilidad del recurso exige que su Platform coincida con la Platform del Product de la ProductOffer vendida.
* Account/Profile guardan solo `isActive`; vistas derivan disponibilidad a partir de habilitación, credenciales y asignaciones. El estado de Subscription se deriva de sus fechas y no requiere job de transición. El vencimiento no devuelve stock automáticamente.
* Platform, Product, ProductOffer, PriceList, User, Seller, Customer y SellerCustomer se inactivan sin cascadas: bloquean la elegibilidad de nuevas operaciones dependientes, pero preservan historial y Subscription vigentes.
* La elegibilidad comercial evalúa conjuntamente actores, relación SellerCustomer, catálogo, precio vigente no ambiguo, inventario, saldo y moneda; no se representa con un único estado.
* Solo una PriceList puede estar activa por `targetRole`. Para una misma lista, oferta y prioridad, PriceListItem no admite intervalos de vigencia solapados; los intervalos consecutivos y prioridades distintas son válidos.
* Una ProductOffer activa es única por `(Product, saleMode)`. Todo OrderItem conserva Product y ProductOffer coherentes entre sí.
* `ProductOffer.saleMode` y `OrderItem.saleModeSnapshot` deben coincidir al crear y confirmar la línea. Desde la primera línea la modalidad de la oferta no cambia. La asignación de cada Subscription conserva esa modalidad: `PROFILE` usa Profile y `FULL_ACCOUNT` usa Account, también en garantía.
* Una Order tiene una única moneda; cada OrderItem la replica. En compras de Wallet, Wallet, Order y WalletTransaction deben usar esa misma moneda dentro de la transacción confirmadora.
* Una compra por Wallet bloquea la Order y la Wallet del buyer, comprueba que la Wallet le pertenece y admite a lo sumo un `PURCHASE_DEBIT` para esa Order. Un reintento concurrente no puede descontar dos veces.
* La relación Seller → Wallet es 1:1 efectiva: Seller y Wallet se crean juntos y la transacción no puede confirmar si falta la Wallet; una Wallet existente no se mueve ni elimina mientras exista su Seller.
* Una Account sin al menos un Profile o sin credenciales requeridas y válidas no es elegible comercialmente, aunque tenga un estado operativo existente.
* La Account no puede perder su último Profile requerido. Desde el primer uso histórico, `Account.platformId` y `Profile.accountId` son inmutables para preservar el significado de las asignaciones.
* Las operaciones que mutan Account, Profile o ResourceAssignmentHistory cruzan una frontera PostgreSQL autorizada: bloquean primero las Accounts raíz por UUID, luego Profiles, historial y Subscriptions existentes, antes de ejecutar DML. El rol de aplicación no recibe DML directo sobre esas tablas.
* Un WalletTransaction puede tener un solo reverso total. El reverso se relaciona con el movimiento original, no con una nueva entidad ni con una modificación del historial.
* Los movimientos de una Wallet tienen una secuencia contable única y creciente. El timestamp complementa la auditoría, pero no determina el orden del ledger ni el saldo vigente.
* “Control de Cuentas” es una consulta operacional sobre órdenes, ítems, suscripciones y asignaciones; no es entidad.

## Payment

[[Payment]] pertenece a la futura compra directa de Customer. Su ciclo de vida y cardinalidad con [[Order]] no están definidos y por ello no forman parte de las cardinalidades cerradas del MVP. No reemplaza el débito de [[WalletTransaction]] del Seller.

## Decisiones pendientes

Las decisiones futuras fuera del MVP, como [[Payment]] para compra directa de Customer, se documentarán al abrir esa fase.
