# Actores del Sistema

El sistema contempla exactamente tres roles:

1. `ADMIN`
2. `SELLER`
3. `CUSTOMER`

---

## ADMIN

ADMIN es un [[User]] con rol `ADMIN`; no existe una entidad `Admin` independiente. Es responsable de gestionar la operación de la plataforma.

### Responsabilidades

- Gestionar usuarios y Sellers.
- Gestionar plataformas de streaming.
- Gestionar inventario.
- Gestionar cuentas y perfiles.
- Gestionar productos y ofertas.
- Gestionar listas de precios.
- Gestionar precios de Seller y Customer.
- Recargar Wallets de los Sellers.
- Consultar movimientos financieros.
- Consultar órdenes y ventas.
- Consultar suscripciones.
- Consultar reportes.
- Cancelar Orders `PENDING` con razón registrada; Orders confirmadas y servicios activos no se alteran.
- Supervisar la operación general del sistema.
- Consultar y administrar credenciales de inventario; cada lectura queda auditada.
- Consultar los eventos de auditoría dentro de su ámbito administrativo; no puede editar ni eliminar esos eventos desde la aplicación.

### Acceso

El Admin tiene acceso administrativo al sistema.

---

## Seller

El Seller es el usuario que compra servicios dentro de la plataforma para posteriormente comercializarlos a sus propios clientes.

### Responsabilidades

- Consultar el catálogo disponible.
- Consultar precios de Seller.
- Consultar disponibilidad.
- Comprar servicios utilizando su Wallet.
- Consultar saldo de Wallet.
- Consultar movimientos de Wallet.
- Consultar historial de compras.
- Vincular/inactivar relaciones con Customers existentes; no crear perfiles Customer.
- Consultar solo Orders, Customers y servicios originados por sus propias compras.
- Leer credenciales únicamente de servicios originados por sus propias Orders; cada lectura queda auditada.
- Consultar próximas expiraciones.
- Consultar métricas de ventas.

### Wallet

El Seller dispone de una Wallet.

La Wallet:

- Puede ser recargada por el Admin.
- Se utiliza para realizar compras.
- Mantiene un historial de movimientos.
- No puede tener saldo negativo.

### Acceso

El Seller no tiene permisos administrativos.

---

## Customer

El Customer es el consumidor final del servicio.

### Responsabilidades

- Iniciar sesión en el MVP.
- Consultar sus propios servicios e historial en modo lectura.
- Leer credenciales de sus propios servicios activos cuando corresponda; cada lectura queda auditada.
- Consultar fechas de expiración.
- Consultar historial de compras.

### Compra directa futura

En el MVP, el Customer no realiza compras directas ni pagos mediante gateway; el portal es de consulta.

En una fase posterior podrá consultar el catálogo, comprar servicios y realizar pagos mediante el gateway que se defina.

### Wallet

El Customer no tiene Wallet.

Las compras directas futuras del Customer se realizarán mediante el sistema de pagos externo.

### Acceso

El Customer solamente puede acceder a la información y operaciones relacionadas con su propia cuenta y sus propios servicios.

---

# Comparación de permisos

| Funcionalidad | Admin | Seller | Customer |
|---|---:|---:|---:|
| Gestionar Users y roles fijos | Crear/editar/inactivar | No | No |
| Gestionar Seller/Customer profiles | Sí | No | No |
| Gestionar catálogo e inventario | Sí | No | No |
| Ver/editar credenciales de inventario | Todos | Solo recursos de sus Orders | Propios servicios activos |
| Consultar Wallet y movimientos | Todas | Propia | No |
| Recargar/ajustar Wallet | Sí | No | No |
| Comprar con Wallet | No | Sí, propia | No |
| Comprar mediante pago externo | Fase futura | No | Fase futura |
| Administrar relación SellerCustomer | Todas | Solo sus relaciones | No |
| Consultar servicios | Todos | Originados por sus Orders | Propios |
| Cancelar Order `PENDING` | Todas | Seller buyer de esa Order | No |
| Cancelar Subscription confirmada | Fuera del MVP | Fuera del MVP | Fuera del MVP |
| Consultar reportes | Todos | Solo métricas propias | No |
| Consultar AuditEvent | Sí, lectura | No | No |

\* El Seller solamente puede gestionar/consultar los clientes que le correspondan.

---

# Regla general de acceso

Cada actor solamente debe poder realizar las operaciones autorizadas para su rol.

Los códigos de rol son fijos (`ADMIN`, `SELLER`, `CUSTOMER`); Admin asigna roles, pero el MVP no define un catálogo dinámico de permisos. El backend valida acceso y ámbito independientemente del frontend. Toda lectura de secreto de Account genera un AuditEvent; los secretos nunca se copian a logs.
