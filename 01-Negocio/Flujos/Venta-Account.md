# Flujo — Venta `FULL_ACCOUNT`

**Caso relacionado:** CU-019; invariantes CU-018, CU-021, CU-033 y CU-035.

1. Admin carga Account con al menos un Profile. Login y secreto pueden quedar pendientes durante el borrador; la Account no es comercialmente elegible hasta que los datos de acceso requeridos estén completos y validados.
2. La disponibilidad se consulta como proyección derivada de `isActive`, credenciales, Profiles y asignaciones actuales.
3. Seller elige ProductOffer `FULL_ACCOUNT`, Account elegible, beneficiario, precio, moneda y cantidad positiva.
4. La transacción confirma Order y líneas, debita Wallet y asigna una Account distinta por unidad. Cada Subscription obtiene una asignación inicial; una asignación completa vuelve no disponibles Account y Profiles mediante la vista.
5. Toda la operación es atómica: falta de saldo, precio/inventario o conflicto devuelve rollback completo y deja Order `PENDING`.

Una garantía reemplaza la Account dentro de la misma Subscription: se registra actor, fecha/razón de falla, se libera la asignación anterior y se crea una nueva cuenta en la misma operación. Expiración/cancelación no reacondicionan el recurso automáticamente. La reincorporación requiere proceso externo y carga/publicación administrativa.
