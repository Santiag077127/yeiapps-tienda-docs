# Alcance del MVP

## Incluido

Los Casos de Uso existentes son la fuente funcional. Seller compra con Wallet y administra servicios de clientes; Customer consulta sus servicios propios, pero no paga directamente.

| Casos de uso | Alcance MVP |
|---|---|
| CU-001 a CU-024 | Incluidos: login; administración de usuarios/perfiles; Seller y Wallet; recargas y ledger; Platform e inventario; catálogo/precio; compra Seller de Profile o Account; historial y Subscription. |
| CU-026 | Incluido: Customer consulta únicamente sus servicios. |
| CU-027 a CU-030 | Incluidos: tableros y reportes operacionales con definiciones de este repositorio. |
| CU-031, CU-032 | Incluidos para Seller: preparar una renovación/continuidad y confirmarla en o después del vencimiento previo, con nueva Order, OrderItem, Subscription y asignación según corresponda. Customer como comprador directo queda futuro. |
| CU-033 a CU-035 | Incluidos: Control de Cuentas, republicación manual de recursos reacondicionados y reposición por garantía. |
| CU-036 | Incluido: vincular/inactivar/reactivar relación SellerCustomer de un Customer ya registrado. |

## Fuera del MVP

- CU-017 para Customer como catálogo orientado a compra directa; Seller sí consulta catálogo.
- CU-025: compra directa de Customer, pago externo y creación de su Order.
- CU-031 y CU-032 con Customer como comprador directo.
- Gateway, Payment, reembolso externo y conciliación.
- Cancelación anticipada de Subscription confirmada. En MVP solo se cancela Order `PENDING`; el período confirmado termina por fecha.
- Recuperación de contraseña, MFA, sesiones persistidas, refresh tokens y permisos dinámicos.
- Multidivisa, automatización de reacondicionamiento/renovación y margen de reventa.

## Futuro contemplado

La compra directa de Customer, pagos externos y sus efectos contables/operativos se diseñarán en una fase posterior. `Payment` es candidata futura y no pertenece al modelo relacional MVP. La lista de precios Customer no habilita por sí sola la compra directa.

## Reglas de alcance que afectan persistencia

- CU-001 incluye Customer como actor de autenticación y consulta; el portal MVP no permite compra directa.
- Customer debe existir antes de vincularlo con Seller mediante SellerCustomer; una Order no crea la relación.
- Compra inmediata confirma completa o no confirma: no hay reserva general ni cumplimiento parcial.
- Una Order de renovación puede permanecer `PENDING` antes del vencimiento; no debita ni reserva recurso. Al confirmar, inicia en la fecha de negocio actual o posterior al término anterior, sin crear Subscription futura.
- Subscription tiene estado derivado por fechas, no estado mutable ni cancelación anticipada en MVP.

## Trazabilidad funcional

Los Casos de Uso constituyen la fuente funcional de referencia para el MVP mientras no exista un backlog formal de Historias de Usuario. La matriz CU → regla de negocio → entidad → atributo → operación → persistencia está en [[Trazabilidad-CU-BD]].