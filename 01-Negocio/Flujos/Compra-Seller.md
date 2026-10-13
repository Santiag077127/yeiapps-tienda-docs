# Compra Seller

1. Seller selecciona oferta, modalidad, recurso y cantidad positiva. Para renovación/continuidad prepara una línea individual con `quantity = 1` y `previousSubscriptionId`.
2. Se crea una Order `PENDING`; snapshots de buyer/beneficiary se inicializan desde User, se refrescan al confirmar y entonces quedan congelados.
3. Antes de confirmar se verifican rol/estado, relación SellerCustomer, oferta/modalidad, precio y moneda vigentes, credenciales, disponibilidad y saldo.
4. Una línea de continuación solo confirma si `businessDate >= previousSubscription.expirationDate`. Antes de esa fecha no se cobra ni se reserva inventario.
5. La transacción bloqueada debita Wallet, confirma la Order, crea exactamente `quantity` Subscriptions y una asignación inicial por cada una. Todo se confirma o se revierte.
6. Si el intento falla, no queda débito, Subscription ni asignación parcial; la Order permanece `PENDING` para corregir o cancelar.

La duración es entero positivo de días en OrderItem. Subscription inicia el día de confirmación y su estado se deriva de las fechas. Vencimiento no libera ni reacondiciona recursos.

Para CU-031, si se reutiliza el mismo recurso, la asignación previa se cierra con motivo `RENEWAL_TRANSFER` y se crea la inicial nueva en la misma transacción. Si se asigna otro recurso, debe estar disponible y el previo permanece retenido para recondicionamiento y publicación manual. El flujo de garantía CU-035 es distinto y conserva Subscription original.

**Auditoría:** se registran confirmación/cancelación, débito, reposición, renovación y cada lectura de credenciales. Seller solo consulta credenciales de servicios originados en sus Orders.