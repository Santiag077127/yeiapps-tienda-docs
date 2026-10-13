# Customer

## Descripción

`Customer` representa al cliente final que recibe y utiliza los servicios de streaming comercializados mediante la plataforma.

Un cliente puede estar relacionado con uno o varios vendedores.

El cliente:

* Puede tener servicios de streaming activos.
* Puede recibir servicios adquiridos por uno o varios vendedores.
* No posee billetera dentro de la plataforma.
* No utiliza el saldo de un vendedor.
* Puede tener múltiples vendedores asociados.
* En una fase posterior podrá acceder directamente a su panel de cliente.

---

## Responsabilidades

`Customer` representa la información específica del cliente dentro del negocio.

Debe permitir:

* Identificar al cliente.
* Asociarlo con un `User`.
* Mantener sus relaciones comerciales con vendedores.
* Identificar los servicios que posee.
* Consultar sus servicios activos.
* Consultar vencimientos.
* Mantener su historial relacionado con los servicios.

`Customer` no debe encargarse directamente de:

* billeteras
* inventario
* administración de precios
* gestión de cuentas de streaming
* procesamiento de pagos
* autenticación

Estas responsabilidades pertenecen a otros componentes del dominio o infraestructura.

---

## Atributos conceptuales

| Atributo    | Descripción                     |
| ----------- | ------------------------------- |
| `id`        | Identificador único del cliente |
| `userId`    | Usuario asociado                |
| `status`    | Estado del cliente              |
| `createdAt` | Fecha de creación               |
| `updatedAt` | Fecha de última actualización   |

Los datos generales de identidad pertenecen a `User`.

---

## Estado

Estados persistidos:

```text id="0q4r8k"
ACTIVE
INACTIVE
```

### ACTIVE

Estado inicial. El cliente puede ser beneficiary de nuevas operaciones si su User y la relación SellerCustomer aplicable están `ACTIVE`.

### INACTIVE

El cliente permanece registrado, pero no puede ser beneficiary de nuevas compras o asignaciones.

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. No cambian órdenes, relaciones comerciales ni Subscription históricas o vigentes.

Toda transición distinta de las anteriores está prohibida. Reactivar Customer no modifica sus operaciones históricas.

---

## Relación con User

Un cliente está asociado a un usuario.

```text id="w5c2n7"
User 1 ─────── 0..1 Customer
```

Esto permite separar:

```text id="p9h3m6"
User
 ├── identidad
 ├── correo
 ├── autenticación
 └── roles

Customer
 ├── información comercial
 └── relaciones con vendedores
```

---

## Relación con Seller

Un cliente puede tener múltiples vendedores.

Un vendedor también puede gestionar múltiples clientes.

Por lo tanto:

```text id="r2v8k4"
Seller N ─────── N Customer
```

La relación se representa mediante:

```text id="a6d1s9"
SellerCustomer
```

Ejemplo:

```text id="m4x7q2"
              ┌── Seller A
              │
Customer X ───┼── Seller B
              │
              └── Seller C
```

Esto significa que la existencia de un cliente no depende de un único vendedor.

---

## SellerCustomer

`SellerCustomer` representa la relación comercial entre un vendedor y un cliente.

Esta entidad/relación será importante porque puede contener información propia de esa relación.

Conceptualmente:

```text id="v8k3p5"
SellerCustomer
 ├── sellerId
 ├── customerId
 ├── status
 ├── createdAt
 └── updatedAt
```

Estados iniciales:

```text id="f7n2m9"
ACTIVE
INACTIVE
```

### ACTIVE

El vendedor puede realizar operaciones para ese cliente.

### INACTIVE

La relación histórica permanece registrada, pero el vendedor no puede utilizarla normalmente para nuevas operaciones.

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. Una relación `ACTIVE` permite operar solo si Seller, Customer y sus User asociados también están `ACTIVE`. Inactivarla no elimina ni cancela órdenes, Subscription ni historial; reactivarla conserva la misma identidad y trazabilidad.

Toda transición distinta de las anteriores está prohibida.

---

## Regla de acceso comercial

Para que un vendedor pueda realizar una compra destinada a un cliente, debe existir una relación válida:

```text id="q5w8e1"
Seller
   │
   ▼
SellerCustomer
   │
   ▼
Customer
```

Esto evita que un vendedor pueda asignar arbitrariamente servicios a cualquier cliente del sistema.

---

## Servicios

Un cliente puede tener múltiples servicios.

```text id="s3d7h2"
Customer
   │
   ├── Subscription
   ├── Subscription
   └── Subscription
```

Cada `Subscription` representa un servicio adquirido/asignado al cliente.

La relación se deriva del beneficiario Customer registrado en Order y en la Subscription originada por OrderItem; no se mantiene una FK directa adicional desde Customer a Subscription.

---

## Vendedor diferente para cada servicio

Debido a que un cliente puede tener varios vendedores, sus servicios pueden provenir de diferentes vendedores.

Ejemplo:

```text id="c8m2v6"
Customer Pedro

   Netflix
      ↑
   Seller A

   Disney+
      ↑
   Seller B

   Spotify
      ↑
   Seller C
```

Por esto, **no debemos colocar `sellerId` directamente como único atributo del cliente**.

El vendedor debe quedar registrado en la relación correspondiente al servicio/orden.

---

## Compra para Customer

Cuando un vendedor compra un servicio para un cliente:

```text id="n4x9r3"
Seller
   │
   │ compra
   ▼
Order
   │
   ├── Buyer = Seller
   │
   └── Beneficiary = Customer
```

El cliente no necesita realizar la compra directamente.

El dinero utilizado proviene de la billetera del vendedor.

---

## Cliente sin billetera

El cliente **no posee `Wallet`**.

```text id="k7p1d5"
Seller ─────── Wallet

Customer ───── X Wallet
```

Si el cliente compra directamente en una fase posterior, el pago se realizará mediante un gateway de pagos.

El cliente no deberá tener saldo interno para adquirir servicios.

---

## Acceso del cliente

En el MVP, Customer puede iniciar sesión y consultar solo sus propios servicios e historial en modo lectura. La compra directa y el catálogo para compra del cliente quedan para una fase posterior.

El panel de cliente podrá mostrar:

* servicios activos
* plataforma
* tipo de servicio
* fecha de inicio
* fecha de vencimiento
* estado
* credenciales cuando corresponda
* información de renovación cuando sea aplicable

Las credenciales deben tratarse como información sensible y no deben exponerse innecesariamente.

---

## Renovaciones

La renovación **no es una propiedad universal del cliente**.

La posibilidad de renovar depende del servicio/producto/perfil adquirido.

Por ejemplo:

```text id="u3p8m1"
Subscription A
 └── renovable: SI

Subscription B
 └── renovable: NO
```

Por lo tanto, `Customer` no debe contener una propiedad como:

```text
canRenew = true
```

La posibilidad de renovación para el cliente pertenece a la configuración explícita del [[Product]] adquirido y se refleja en el contexto del servicio. No se deduce de la cuenta, el perfil ni su historial.

La plataforma deberá informar al vendedor cuando un servicio:

* sea renovable
* no sea renovable
* esté próximo a vencer
* haya vencido

La regla se configura en `Product`; la renovación confirmada crea nuevo `OrderItem` y `Subscription` enlazados mediante `previousSubscriptionId`.

---

## Historial

La información histórica del cliente debe conservarse.

No se deben eliminar físicamente registros importantes relacionados con:

* órdenes
* servicios
* pagos
* asignaciones
* renovaciones
* relaciones comerciales

Si un cliente deja de utilizar la plataforma, su estado puede cambiar.

```text id="h6q2w9"
ACTIVE
   ↓
INACTIVE
```

La información histórica permanece disponible.

---

## Reglas de negocio relacionadas

* Un cliente debe estar asociado a un `User`.
* El User asociado debe conservar el rol `CUSTOMER` mientras exista el perfil Customer; la alta se realiza atómicamente con ese rol.
* Un cliente puede tener múltiples vendedores.
* Un vendedor puede tener múltiples clientes.
* La relación vendedor-cliente debe ser explícita.
* Un vendedor solo puede comprar para un cliente con el que tenga una relación válida.
* Un cliente no posee billetera.
* Un cliente puede tener múltiples servicios.
* Un cliente puede recibir servicios adquiridos por diferentes vendedores.
* El vendedor que originó una compra debe quedar registrado históricamente.
* La posibilidad de renovación para el cliente depende de la configuración explícita del producto adquirido.
* Los servicios vencidos deben conservarse históricamente.
* Los registros históricos del cliente no deben eliminarse físicamente.

---

## Relaciones principales

```text id="y8f4k2"
                    ┌──────────────┐
                    │     User     │
                    └──────┬───────┘
                           │
                           │
                  ┌────────▼────────┐
                  │    Customer     │
                  └────────┬────────┘
                           │
                    N      │      N
                           │
                  ┌────────▼────────┐
                  │ SellerCustomer  │
                  └────────┬────────┘
                           │
                           │
                           ▼
                         Seller
```

Y posteriormente:

```text id="b5r9c3"
Customer
   │
   └── Subscription
          │
          └── Account / Profile
```

---

## Fuera del alcance de Customer

No pertenece a `Customer`:

* saldo
* billetera
* movimientos financieros
* inventario global
* precios del catálogo
* cuentas de streaming
* perfiles de streaming
* lógica del gateway de pagos
* lógica de autenticación

---

## Ejemplos

Pedro puede estar relacionado con Seller A, Seller B y Seller C. Una [[Subscription]] de cada plataforma puede conservar el origen comercial de su orden, sin que `Customer` tenga un único `sellerId` ni una billetera.

---

## Estado del diseño

**Definido**

Pendiente de implementación técnica:

* tipo de `CustomerId`
* estructura definitiva de `SellerCustomer`
* información adicional del cliente
* reglas de bloqueo
* restricciones de persistencia
* índices
* auditoría
