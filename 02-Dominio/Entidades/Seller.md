# Seller

## Descripción

`Seller` representa al vendedor/distribuidor que utiliza la plataforma para comprar servicios de streaming y posteriormente comercializarlos a sus clientes.

El `Seller` tiene una relación directa con un `User`, pero contiene información y comportamiento específico del negocio de distribución.

El vendedor:

* Tiene una billetera.
* Compra servicios utilizando el saldo de su billetera.
* Puede comprar servicios para sí mismo.
* Puede comprar servicios para uno de sus clientes.
* Puede tener múltiples clientes.
* Puede participar en la atención de un cliente que también tenga otros vendedores.
* Consulta su historial de compras.
* Consulta sus movimientos de billetera.
* Consulta información relacionada con sus servicios y clientes.

---

## Responsabilidades

`Seller` debe representar el contexto comercial del vendedor.

Entre sus responsabilidades están:

* Identificar al vendedor dentro del negocio.
* Mantener su relación con `User`.
* Asociar los clientes que gestiona.
* Mantener acceso a su billetera.
* Permitir identificar quién realizó una compra.
* Permitir determinar para quién fue realizada una compra.
* Consultar información comercial propia.

`Seller` **no debe administrar directamente el inventario global**.

El inventario pertenece a la plataforma y el vendedor obtiene servicios mediante una compra.

---

## Atributos conceptuales

| Atributo    | Descripción                      |
| ----------- | -------------------------------- |
| `id`        | Identificador único del vendedor |
| `userId`    | Usuario asociado                 |
| `status`    | Estado del vendedor              |
| `createdAt` | Fecha de creación                |
| `updatedAt` | Fecha de última actualización    |

Los datos personales como nombre, correo y credenciales pertenecen inicialmente a `User`.

La información comercial específica del vendedor puede crecer posteriormente si el negocio lo requiere.

---

## Estado

Estados persistidos:

```text id="7w0g7m"
ACTIVE
INACTIVE
```

### ACTIVE

Estado inicial. El vendedor puede acceder a sus funcionalidades comerciales si su User también está `ACTIVE`.

Puede:

* consultar saldo
* realizar compras
* consultar clientes
* consultar servicios
* consultar movimientos

### INACTIVE

El vendedor permanece registrado pero no puede realizar nuevas compras ni operaciones comerciales.

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. La reactivación exige que su User esté `ACTIVE` y no altera órdenes, relaciones, Subscription ni historial.

Toda transición distinta de las anteriores está prohibida. La elegibilidad de compra también requiere Wallet con saldo suficiente y catálogo, precio e inventario elegibles.

---

## Relación con User

Cada vendedor debe estar asociado a un usuario.

```text id="z3w5k2"
User 1 ─────── 0..1 Seller
```

No todos los `User` son vendedores.

Ejemplo:

```text id="l3b4g5"
User
 ├── ADMIN
 ├── SELLER → Seller
 └── CUSTOMER → Customer
```

La identidad pertenece a `User`; la información comercial pertenece a `Seller`.

---

## Relación con Customer

Un vendedor puede gestionar múltiples clientes.

Sin embargo, un cliente **no necesariamente pertenece exclusivamente a un vendedor**.

Por lo tanto, la relación es muchos a muchos:

```text id="qk9l1r"
Seller N ─────── N Customer
```

Se utilizará una relación intermedia conceptual:

```text id="t8r2p6"
SellerCustomer
```

Esta relación permitirá representar:

```text id="9x3m1z"
Seller A ──┐
           ├── Customer X
Seller B ──┤
           └── Customer Y
```

Y:

```text id="5d8q2v"
Customer X
 ├── Seller A
 ├── Seller B
 └── Seller C
```

Esto es importante porque el cliente puede recibir servicios adquiridos mediante diferentes vendedores.

---

## Billetera

Cada vendedor posee una billetera.

```text id="1h7k4p"
Seller 1 ─────── 1 Wallet
```

La billetera permite al vendedor comprar productos dentro de la plataforma.

El vendedor no puede realizar una compra si el saldo disponible es insuficiente.

La billetera tendrá:

* saldo actual
* historial de movimientos
* recargas realizadas por el administrador
* débitos generados por compras

La lógica financiera detallada pertenece al módulo `wallet`.

---

## Compras

El vendedor puede realizar una compra para:

### 1. Sí mismo

```text id="e4p7s2"
Seller
  │
  └── Order
       └── Beneficiary: Seller
```

### 2. Uno de sus clientes

```text id="m8q3v1"
Seller
  │
  └── Order
       └── Beneficiary: Customer
```

Por lo tanto, debemos diferenciar:

```text id="c2f9w6"
Buyer
```

de:

```text id="r5n1k8"
Beneficiary
```

Ejemplo:

```text id="x7b3q9d"
Buyer:
    Seller Juan

Beneficiary:
    Customer Pedro
```

El dinero sale de la billetera de Juan, pero el servicio queda destinado a Pedro.

---

## Regla importante

El vendedor solo debería poder comprar para un cliente con el que tenga una relación válida.

```text id="n6v2c8"
Seller
   │
   ├── Customer A ✓
   ├── Customer B ✓
   └── Customer C ✗
```

Si el cliente no está asociado al vendedor, la operación debe rechazarse o requerir primero la creación de la relación correspondiente.

---

## Vendedor comprando para sí mismo

El vendedor también puede adquirir servicios para uso propio.

En este caso no necesitamos crear artificialmente un `Customer`.

```text id="k4m8p2"
Buyer       = Seller
Beneficiary = Seller
```

Esto mantiene el modelo más limpio.

---

## Servicios del vendedor

Los servicios que el vendedor adquiere pueden terminar en dos escenarios:

```text id="r9c3v7"
Seller
  │
  ├── Servicio para sí mismo
  │
  └── Servicio para Customer
```

La asignación final del servicio se registrará mediante `Subscription`.

`Seller` no debe guardar directamente todas las credenciales o datos del servicio.

---

## Dashboard del vendedor

El vendedor tendrá acceso a información como:

* saldo actual
* compras realizadas
* movimientos de billetera
* servicios activos
* próximos vencimientos
* clientes gestionados
* servicios vendidos
* información de ventas
* historial comercial

El dashboard es una responsabilidad de aplicación/presentación.

`Seller` no debe contener lógica de presentación ni consultas específicas del dashboard.

---

## Reglas de negocio relacionadas

* Un vendedor debe estar asociado a un `User`.
* El User asociado debe conservar el rol `SELLER` mientras exista el perfil Seller; la alta se realiza atómicamente con ese rol.
* Un vendedor debe tener una única billetera.
* User con rol `SELLER`, Seller y Wallet con saldo inicial cero se crean en una sola transacción. Si falla cualquiera de ellos, no persiste el alta; no existe una segunda operación manual para crear la Wallet.
* El vendedor utiliza su billetera para realizar compras.
* El vendedor no puede gastar más saldo del disponible.
* El administrador puede recargar la billetera del vendedor.
* Los movimientos financieros no deben eliminarse físicamente.
* Un vendedor puede tener múltiples clientes.
* Un cliente puede estar relacionado con múltiples vendedores.
* Un vendedor puede comprar para sí mismo.
* Un vendedor puede comprar para un cliente relacionado.
* El comprador de una orden y el beneficiario de la orden pueden ser personas diferentes.
* La compra debe conservar el vendedor que realizó la operación.
* El historial de compras del vendedor debe conservarse aunque posteriormente cambie el estado del vendedor.
* Inactivar Seller no liquida, transfiere ni confisca su saldo de Wallet; saldo e historial se preservan.

---

## Relaciones principales

```text id="d7p4k9"
                 ┌──────────────┐
                 │     User     │
                 └──────┬───────┘
                        │
                        │ 1
                        │
                        │ 0..1
                 ┌──────▼───────┐
                 │    Seller    │
                 └──────┬───────┘
                        │
              ┌─────────┼─────────┐
              │         │         │
              │         │         │
              ▼         ▼         ▼
           Wallet   SellerCustomer  Order
                        │            │
                        ▼            ▼
                    Customer     OrderItem
```

---

## Dependencias conceptuales

`Seller` se relaciona principalmente con:

* `User`
* `Customer`
* `Wallet`
* `Order`
* `Subscription`

Pero no debe depender directamente de:

* PostgreSQL
* JPA
* Spring
* controladores HTTP
* gateways de pago

Estas dependencias pertenecen a las capas externas.

---

## Ejemplos

Juan, como seller, compra dos servicios en una misma [[Order]]. Puede ser beneficiario de uno y asignar el otro a Pedro solamente si existe una relación `SellerCustomer` activa con Pedro. El débito se realiza siempre desde la billetera de Juan.

---

## Fuera del alcance

No contiene el inventario global, los precios de catálogo, las credenciales de streaming, la lógica de presentación del dashboard ni detalles de infraestructura.

---

## Estado del diseño

**Definido**

Pendiente de implementación técnica:

* tipo de `SellerId`
* estructura de `SellerCustomer`
* reglas de alta/baja del vendedor
* campos comerciales adicionales
* restricciones de persistencia
* índices
* auditoría
