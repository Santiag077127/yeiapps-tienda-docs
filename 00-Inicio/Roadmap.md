# Roadmap documental y de datos

## Fase 0 — Especificación previa a BD

Alcance, entidades, reglas, estados calculados, relaciones, seguridad, auditoría y trazabilidad CU están cerrados. La documentación queda lista para revisar y aprobar el modelo lógico antes de escribir DDL.

## Fase 1 — MVP con Wallet

Implementar User/roles/perfiles, SellerCustomer, Wallet/ledger, catálogo/precios, inventario Account/Profile, Orders/OrderItems, Subscriptions, historial de asignaciones, auditoría y consultas operacionales. La especificación física documental es [[Modelo-Fisico-PostgreSQL]]; no se ha creado DDL.

## Fase 2 — Compra directa de Customer

Definir gateway, Payment, callbacks, idempotencia, moneda, errores/reintentos, reembolsos, relación Payment–Order y seguridad de credenciales del proveedor. CU-025 y la parte de comprador directo de CU-031/032 quedan en esta fase.

## Orden de trabajo siguiente

1. Crear/revisar el modelo lógico definitivo contra [[Modelo-ER]], [[Relaciones]] y [[Trazabilidad-CU-BD]].
2. Aprobar el modelo lógico y convertir la especificación física en DDL PostgreSQL.
3. Implementar cambios Liquibase por orden de dependencia; crear constraints cíclicas/diferidas, triggers, funciones y permisos después de las tablas base.
4. Implementar vistas de disponibilidad/estado derivado e índices.
5. Configurar moneda, zona horaria y contrato de credenciales de cada Platform antes de las operaciones del despliegue.