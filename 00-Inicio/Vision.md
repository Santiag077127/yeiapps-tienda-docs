# Visión

## Propósito

Streaming Business System administra la comercialización de servicios de streaming mediante Sellers, con inventario compartido, catálogo y listas de precios. El Seller paga las compras desde una Wallet y puede destinar el servicio a sí mismo o a uno de sus Customers relacionados. El sistema conserva la operación comercial, el período de servicio, la asignación del recurso y el ledger financiero.

## Actores

- **ADMIN:** administra identidad, catálogo, precios, inventario, recargas, operaciones y reportes.
- **SELLER:** consulta disponibilidad, compra con Wallet, gestiona sus relaciones comerciales con Customers y consulta sus servicios y movimientos.
- **CUSTOMER:** en el MVP puede iniciar sesión y consultar únicamente sus propios servicios e historial. No compra directamente ni tiene Wallet.

Los roles son códigos cerrados: `ADMIN`, `SELLER` y `CUSTOMER`. Un User puede tener más de un rol; `Seller` y `Customer` son perfiles de negocio vinculados a User.

## Límites del producto

El MVP cubre la compra financiada por Wallet, el inventario de Account/Profile, catálogo, períodos de servicio, reposiciones por garantía, consulta de servicios, administración y reportes operacionales. La compra directa de Customer con pasarela no pertenece al MVP.

## Fuente funcional

Los Casos de Uso constituyen la fuente funcional de referencia para el MVP mientras no exista un backlog formal de Historias de Usuario. Las reglas de negocio y los documentos de dominio precisan las invariantes de persistencia.
