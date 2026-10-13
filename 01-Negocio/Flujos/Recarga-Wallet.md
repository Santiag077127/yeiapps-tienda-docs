# Flujo — Recarga de Wallet

**Casos relacionados:** CU-006, CU-007, CU-008, CU-030.

1. Admin selecciona Seller y Wallet activa para operación; consulta titular, saldo y moneda.
2. Admin indica monto positivo en la moneda única configurada para la instalación y confirma.
3. La operación valida User/rol `ADMIN`, Seller/User activos, moneda configurada y valor válido; serializa la Wallet bajo bloqueo.
4. Se inserta `WalletTransaction` `TOP_UP`, con actor, secuencia, saldo anterior, delta y saldo resultante; Wallet cambia solo como efecto del ledger.
5. Se agrega AuditEvent con actor, acción y referencia segura. El movimiento financiero es la fuente canónica del monto.

No se modifica ni elimina un movimiento anterior. Un ajuste/reverso posterior es un movimiento separado autorizado. La moneda concreta es configuración de despliegue y debe definirse antes de la primera Wallet; no se asume COP.
