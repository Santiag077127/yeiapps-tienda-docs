# Arquitectura General

## Alcance

La arquitectura documentada es lógica y no selecciona framework, proveedor cloud ni protocolo de API que los Casos de Uso no hayan especificado. Componentes funcionales: cliente web, servicio de aplicación/API, PostgreSQL y Liquibase para cambios de esquema. Los repositorios `store-frontend`, `store-service` y `store-db` se mantienen separados.

## Límites y propiedad de datos

- **Identidad:** User, UserRole y perfiles Seller/Customer; roles de negocio fijos.
- **Catálogo:** Platform, Product, ProductOffer, PriceList y PriceListItem.
- **Inventario:** Account, Profile y ResourceAssignmentHistory; disponibilidad calculada, no stock duplicado.
- **Ventas y servicio:** Order, OrderItem y Subscription; snapshots congelados al confirmar.
- **Finanzas:** Wallet y WalletTransaction; el ledger es canónico para movimientos y saldo.
- **Auditoría transversal:** AuditEvent append-only, sin secretos.

## Transacciones de negocio

La confirmación Seller se ejecuta en una transacción PostgreSQL única con locks y revalidación de Order/ítems, catálogo/precio, inventario y Wallet. Débito, confirmación, exactamente `quantity` períodos y asignaciones quedan atómicos. Funciones autorizadas son la frontera de escritura para inventario; la aplicación no recibe DML directo a Account, Profile o su historial.

Recarga, garantía, cancelación de Order PENDING, renovación y expiración son operaciones distintas y auditables. El estado de Subscription se deriva por fechas; el vencimiento no libera inventario. No hay cancelación anticipada de Subscription confirmada en el MVP.

## Seguridad transversal

`password_hash` es hash unidireccional. Secretos de streaming se cifran antes de persistir con clave fuera de PostgreSQL; la carga útil debe identificar versión/suite, key ID, nonce, ciphertext y tag. Metadatos JSONB no contienen secretos y se validan por plataforma. Toda lectura de credenciales genera AuditEvent. Nunca registrar credenciales, hashes o tokens en logs.

La autorización se aplica en backend por rol y ámbito del registro; ocultar controles de UI no es autorización. Seller ve solo sus Orders/relaciones/servicios; Customer solo lo propio; Admin tiene acceso operativo general. No se modelan sesiones, refresh tokens ni recuperación de contraseña en este MVP.

## Datos de instalación

Cada instalación configura un código ISO 4217 único y `store.business_timezone` antes de operar. Son parámetros externos del despliegue, no columnas ni bloqueos del diseño. La fecha de negocio calculada por esa zona gobierna Subscription y reportes; no hay job que actualice estado de expiración.

## Estado de implementación

El modelo y el flujo están documentados; no hay tablas, funciones, endpoints ni cambios Liquibase de negocio implementados todavía. La secuencia documental de DDL está en el README de `yeiapp-store-db`.
