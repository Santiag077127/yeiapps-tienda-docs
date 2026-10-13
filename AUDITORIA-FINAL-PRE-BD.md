# Auditoría final previa a BD

**Fecha:** 2026-10-13  
**Alcance:** repositorios `yeiapps-tienda-docs` y `yeiapp-store-db`, con referencias funcionales, de dominio, arquitectura y persistencia.  
**Estado:** 🟢 **LISTO PARA COMENZAR BD**

## Veredicto

La especificación del MVP permite iniciar el modelo conceptual definitivo, el modelo lógico y la revisión física PostgreSQL sin decisiones de negocio pendientes que bloqueen el esquema. El próximo paso es revisar y aprobar el modelo lógico; después se podrá convertir la especificación física a DDL y migraciones Liquibase.

La documentación fue reconciliada desde los CU, no se inventaron HUs y cada CU-001 a CU-036 tiene clasificación/trazabilidad. El repositorio de base de datos continúa como scaffold.

**No se creó DDL, tabla, SQL ejecutable, migración ni cambio Liquibase. No se ejecutaron pruebas ni migraciones.** Los changelogs de tablas y demás componentes permanecen vacíos.

## Bloqueadores restantes

**Ninguno identificado para el diseño conceptual, lógico o físico del MVP.** La moneda de instalación, zona horaria y contratos de credenciales son configuración necesaria antes de operar, pero no cambian las tablas ni relaciones. La retención legal se debe fijar antes de producción como política operativa.

## Decisiones cerradas que impactan BD

- **Alcance:** Seller compra con Wallet y administra sus clientes; Customer consulta sus servicios. Customer como comprador, Payment/gateway y sus reembolsos son fase futura. Los CU constituyen la fuente funcional oficial mientras no haya backlog formal de Historias de Usuario; no se inventaron HUs.
- **User y snapshots:** User es fuente vigente de nombre obligatorio, email obligatorio/único y teléfono opcional. Order copia buyer/beneficiary al crear y refresca al confirmar; después los snapshots son inmutables. Cambios posteriores de User no alteran la historia.
- **Wallet:** una Wallet 1:1 por Seller, saldo inicial cero, moneda ISO única por despliegue; Seller no puede cambiarla. Recargas/ajustes son de Admin, débito de compra solo desde Wallet del buyer, no hay saldo negativo. WalletTransaction inmutable conserva delta, saldos, secuencia, actor, momento y referencias. CU-008 presenta `REVERSAL` en su fila y deriva `REVERSED`/`NOT_REVERSED` para el original; no tiene status persistido.
- **OrderItem y cantidad:** quantity es entero positivo de unidades iguales en condiciones comerciales. La confirmación produce exactamente una Subscription y una asignación inicial por unidad. Si una unidad falla, rollback total; Order sigue PENDING, sin débito ni cumplimiento parcial. La línea de renovación/continuidad apunta a un período previo y exige quantity 1.
- **Subscription:** serviceDurationDays entero positivo vive en OrderItem porque es el término comercial común de la línea; no se duplica. Subscription persiste start_date y expiration_date con intervalo `[start_date, expiration_date)`; `expiration_date = start_date + service_duration_days`. Nace al confirmar. ACTIVE/EXPIRED se deriva por `businessDate`; no se persiste status, no hay estado Subscription PENDING/CANCELLED ni job diario de transición. Vencer no libera recurso.
- **Renovación/continuidad:** OrderItem conserva previous_subscription_id en el borrador; Subscription confirmada copia el vínculo. Confirma solo si `businessDate >= expirationDate` del período previo y usa `startDate = max(expirationDate anterior, businessDate)`. No hay cobro, recurso reservado ni período de servicio futuro. Product y beneficiario deben coincidir. Un período admite un sucesor confirmado. Si reutiliza recurso, se transfiere atómicamente; si se usa otro, el nuevo recurso debe estar disponible y el anterior queda retenido hasta reacondicionamiento/publicación manual.
- **Asignaciones/garantía:** ResourceAssignmentHistory conserva recurso y secuencia, actor de asignación, datos mínimos de falla, actor/motivo/fecha de liberación. La fila anterior identifica el recurso fallido; la nueva, su reemplazo. INITIAL_ASSIGNMENT es obligatorio; WARRANTY_REPLACEMENT mantiene la Subscription. RAH más AuditEvent bastan; no se agrega Incident.
- **Inventario:** Account/Profile persisten `is_active`; vistas derivan disponibilidad de asignaciones, credenciales y relación padre/hijo. Estados de Account/Profile no duplican estado de Subscription. Expiración no libera stock; liberación por garantía, transferencia o reacondicionamiento manual requiere actor y motivo.
- **Credenciales y autenticación:** `password_hash` es adaptativo, nunca se guarda contraseña en claro. Secretos de streaming se cifran como envelope y la clave queda fuera de PostgreSQL. `credential_metadata` solo admite JSONB no secreto. Toda lectura de credencial genera auditoría. No se persisten sesiones, refresh tokens ni claves criptográficas en MVP.
- **Auditoría/cancelación:** AuditEvent append-only cubre cambios críticos, actor, hora, objeto, acción, cambios no sensibles y razón; no contiene secretos. WalletTransaction/RAH son canónicos para sus dominios. Seller comprador o Admin cancela Order solo mientras PENDING; AuditEvent registra razón/actor/hora, por lo que no se duplican campos de cancelación. Una Order confirmada es histórica; la cancelación anticipada de Subscription no está en MVP.
- **Reportes:** ventas y compras se filtran por `confirmedAt`; movimientos por `occurredAt`; expiraciones por rango explícito de `expirationDate`. Venta Seller = Order CONFIRMED con Customer relacionado; compra = Order CONFIRMED del Seller como buyer. Bruto = `sum(quantity × unitPrice)` por moneda, sin representar margen o ingreso contable. Los reportes son consultas del backend; no se requieren tablas/materializaciones.
- **Normalización:** 1FN conserva valores escalares y filas por movimiento/línea/asignación; 2FN usa claves y tablas puente explícitas; 3FN separa fuentes por entidad. Redundancias intencionales: snapshots; Wallet.balance protegido contra ledger; moneda copiada para historia; previous_subscription_id copiado entre línea pendiente y período confirmado, con igualdad transaccional. Stock y estado de Subscription son derivados.

## Entidades definitivas del MVP

19 tablas objetivo:

- **identity (6):** app_user, user_role, audit_event, seller, customer, seller_customer.
- **catalog (5):** platform, product, product_offer, price_list, price_list_item.
- **inventory (3):** account, profile, resource_assignment_history.
- **sales (3):** orders, order_item, subscription.
- **wallet (2):** wallet, wallet_transaction.

`Payment`, Admin como entidad, Stock, Renewal, Incident, Guarantee, Replacement, Reconditioning, sesiones y tablas de reporte no forman parte de las tablas del MVP. Payment está documentada como fase futura y no aparece en el ER MVP.

## Relaciones definitivas

- User 1:0..1 Seller y User 1:0..1 Customer; User N:N UserRole con códigos cerrados `ADMIN`, `SELLER`, `CUSTOMER`.
- Seller N:N Customer mediante SellerCustomer.
- Seller 1:1 Wallet; Wallet 1:N WalletTransaction.
- Platform 1:N Account y 1:N Product; Account 1:N Profile.
- Product 1:N ProductOffer; PriceList N:N ProductOffer mediante PriceListItem.
- Seller 1:N Order como buyer; cada Order tiene buyer Seller y exactamente un beneficiary Seller o Customer, excluyentes.
- Order 1:N OrderItem; OrderItem 1:N Subscription (al confirmar, el conteo es exactamente quantity).
- OrderItem y Subscription conservan la referencia al período anterior en borrador y éxito; Subscription tiene como máximo un sucesor confirmado.
- Subscription 1:N ResourceAssignmentHistory; cada asignación apunta a exactamente un recurso: Account (`FULL_ACCOUNT`) o Profile (`PROFILE`).

## Estados definitivos

- Entidades de identidad/catálogo/relación: `ACTIVE`, `INACTIVE`.
- Order: `PENDING`, `CONFIRMED`, `CANCELLED`; transiciones PENDING→CONFIRMED/CANCELLED.
- Subscription: estado calculado `ACTIVE`/`EXPIRED`; no se almacena.
- Account: disponibilidad derivada `AVAILABLE`, `PARTIALLY_ASSIGNED`, `FULLY_ASSIGNED`, `INACTIVE`.
- Profile: disponibilidad derivada `AVAILABLE`, `ASSIGNED`, `BLOCKED_BY_FULL_ACCOUNT`, `INACTIVE`.
- WalletTransaction: tipos `TOP_UP`, `PURCHASE_DEBIT`, `ADJUSTMENT_CREDIT`, `ADJUSTMENT_DEBIT`, `REVERSAL`; estado mostrado derivado, no persistido.
- ResourceAssignmentHistory: reason de alta `INITIAL_ASSIGNMENT`/`WARRANTY_REPLACEMENT`; motivo de liberación `WARRANTY_REPLACEMENT`/`RENEWAL_TRANSFER`/`MANUAL_RECONDITIONING`.

## Reglas críticas de integridad

PK/FK con borrado restrictivo; snapshots y Orders confirmadas inmutables; precio/oferta/modalidad y moneda revalidados en confirmación; una Wallet y saldo no negativo; ledger ordenado por secuencia monotónica y sin mutación; compra idempotente; cantidad y fulfillment atómicos; Subscription siempre con asignación inicial al COMMIT; un sucesor confirmado por período; exclusiones temporales para Profile y Account FULL_ACCOUNT; bloqueo de Account antes de Profile/RAH para concurrencia; disponibilidad derivada; liberación de RAH con actor/razón en una única actualización; credenciales cifradas con clave externa; AuditEvent append-only.

## Inventario físico e implementación

- **19 tablas objetivo.**
- **66 estructuras indexadas:** 19 PK + 18 UNIQUE adicionales + 3 EXCLUDE + 26 índices explícitos. El inventario está en [[Modelo-Fisico-PostgreSQL]].
- `yeiapp-store-db/README.md` documenta dependencias y el orden de changesets. La FK OrderItem→Subscription se agrega después de las tablas base para resolver el ciclo con Subscription→OrderItem; constraints diferidos, funciones, triggers, vistas e índices siguen fases separadas.
- Los 20 changelogs hoja continúan vacíos; los cuatro archivos agregadores solo incluyen esas familias, sin changesets ejecutables.

## Decisiones que requieren intervención antes de operar o fuera del MVP

No bloquean el diseño de la BD:

- El operador debe fijar `store.currency_code` y `store.business_timezone` antes de operar; no se asume COP ni zona por usuario.
- Cada Platform requiere contrato de credenciales no secretas y validador antes de publicar inventario de esa plataforma.
- La política legal de conservación/archivo/pseudonimización se fija antes de producción. MVP no hace purga automática y el rol de aplicación no modifica AuditEvent.
- Compra directa de Customer, Payment/gateway, cancelación anticipada de Subscription, recuperación de password, MFA, sesiones persistidas, multidivisa, margen de reventa y automatización de renovación/reacondicionamiento están marcados FASE FUTURA o FUERA DEL MVP.

## Archivos y áreas actualizadas

- Inicio y alcance: `README.md`, `00-Inicio/README.md`, `Alcance-MVP.md`, `Vision.md`, `Roadmap.md`.
- Negocio: `Actores.md`, `Casos-de-Uso.md`, `Reglas-de-Negocio.md` y flujos `Compra-Seller.md`, `Compra-Customer.md`, `Recarga-Wallet.md`, `Venta-Account.md`.
- Dominio: `Mapa-del-Dominio.md`; entidades `User`, `Customer`, `Order`, `OrderItem`, `Subscription`, `Wallet`, `WalletTransaction`, `Account`, `Profile`, `ResourceAssignmentHistory`, `AuditEvent` y `Payment`.
- Arquitectura y BD: `Arquitectura-General.md`, `Modulos.md`, `Modelo-ER.md`, `Relaciones.md`, `Modelo-Fisico-PostgreSQL.md`, `Trazabilidad-CU-BD.md`.
- Desarrollo/repositorio DB: `Backlog.md`, `Decisiones-Pendientes.md`, `yeiapp-store-db/README.md` y este informe.

## Inconsistencias restantes

No se identificaron inconsistencias conocidas que bloqueen el diseño. La revisión estática verificó 40 archivos Markdown no vacíos, cero enlaces wiki rotos, los 36 CU, las 19 tablas, 66 estructuras indexadas (19 PK + 18 UNIQUE + 3 EXCLUDE + 26 explícitas) y 20 changelogs hoja vacíos con cuatro agregadores sin changesets. La validación final no implementa ni ejecuta DDL.

## Próximo paso

> Crear/revisar el modelo lógico y posteriormente generar el DDL PostgreSQL y las migraciones Liquibase.
