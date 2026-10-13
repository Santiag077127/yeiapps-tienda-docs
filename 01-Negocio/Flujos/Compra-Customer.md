# Flujo futuro — Compra directa de Customer

**Caso relacionado:** CU-025, fuera del MVP. No se define DDL ni integración en esta fase.

Las etapas generales de consultar catálogo Customer, seleccionar oferta, conocer precio, iniciar y confirmar pago, asignar recurso y crear Order/Subscription son una intención de alto nivel del CU. Antes de incluirla hay que decidir proveedor, estados, callbacks, idempotencia, errores/reintentos, reembolsos, relación Payment–Order, moneda y seguridad de credenciales del gateway. La compra futura no otorgará Wallet a Customer.

Hasta cerrar esas decisiones, Customer solo consulta sus propios servicios e historial. `Payment` no se crea en el modelo físico MVP.
