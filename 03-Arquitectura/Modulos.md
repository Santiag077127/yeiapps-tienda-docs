# Módulos del sistema

| Módulo | Responsabilidad | Elementos principales | Dependencias |
|---|---|---|---|
| Identity | Autenticación de User, roles y perfiles comerciales | User, UserRole, Seller, Customer, SellerCustomer | Auditoría, ventas |
| Catalog | Plataformas, productos, ofertas y precios vigentes | Platform, Product, ProductOffer, PriceList, PriceListItem | Identity, ventas |
| Inventory | Alta/consulta de recursos, disponibilidad y asignación/reemplazo | Account, Profile, ResourceAssignmentHistory, vistas | Catalog, Subscription |
| Store/Sales | Orders/ítems, snapshot, confirmación y consultas de compras | Order, OrderItem | Identity, Catalog, Inventory, Wallet |
| Wallet | Saldo, recargas, débitos, ajustes y reversos | Wallet, WalletTransaction | Seller, Order, Auditoría |
| Subscription | Períodos y consultas; estado ACTIVE/EXPIRED derivado por fechas | Subscription | OrderItem, Inventory |
| Reporting | Métricas por rango y consultas operativas | Vistas/consultas sobre Orders, WalletTransaction, Subscription y asignaciones | Todos los módulos |
| Audit | Trazabilidad transversal incluida lectura de credenciales | AuditEvent | Identidad y operaciones de escritura/lectura sensibles |

## Límites

- `Payment` solo pertenece a la futura compra directa Customer (CU-025); no existe en el esquema MVP.
- Renovar crea nueva Order/OrderItem/Subscription y conserva el enlace al período anterior. Garantía modifica únicamente ResourceAssignmentHistory de la Subscription vigente.
- Account/Profile availability se deriva; no se mantiene un stock independiente ni estados duplicados.
- Las restricciones de transacción/autorización se definen en [[Modelo-Fisico-PostgreSQL]], no se duplican como reglas diferentes por módulo.
