# User

## Descripción

`User` representa la identidad de una persona que puede acceder al sistema.

Un usuario puede desempeñar uno o varios roles dentro de la plataforma:

* ADMIN
* SELLER
* CUSTOMER

La entidad `User` representa la identidad y autenticación. Las responsabilidades específicas de cada tipo de usuario se modelan en sus respectivos módulos y entidades.

---

## Responsabilidades

`User` debe permitir:

* Identificar de forma única a una persona dentro del sistema.
* Gestionar sus datos básicos.
* Mantener sus credenciales de acceso.
* Determinar si el usuario está activo.
* Asociar uno o varios roles.
* Mantener información necesaria para auditoría.

`User` **no debe contener lógica específica de negocio** relacionada con:

* billeteras
* compras
* inventario
* suscripciones
* precios
* pagos

Estas responsabilidades pertenecen a otros elementos del dominio.

---

## Atributos conceptuales

| Atributo    | Descripción                           |
| ----------- | ------------------------------------- |
| `id`        | Identificador único del usuario       |
| `name`      | Nombre del usuario                    |
| `email`     | Correo electrónico                    |
| `passwordHash` | Hash unidireccional de la contraseña; nunca se persiste la contraseña en claro. |
| `phone` | Teléfono opcional normalizado, usado para contacto y snapshots. |
| `status`    | Estado del usuario                    |
| `createdAt` | Fecha de creación                     |
| `updatedAt` | Fecha de última actualización         |

---

## Estado

Estados persistidos:

```text
ACTIVE
INACTIVE
```

### ACTIVE

Estado inicial. El usuario puede iniciar sesión y utilizar las funcionalidades permitidas por sus roles y estados comerciales.

### INACTIVE

El usuario existe e historial permanece, pero no puede autenticarse ni acceder a funcionalidades protegidas.

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. Los bloqueos temporales por intentos fallidos, recuperación de contraseña u otros controles de seguridad no son estados de negocio persistidos de User.

Toda transición distinta de las anteriores está prohibida. Reactivar User recupera el acceso sujeto a sus roles y a los estados comerciales vigentes de Seller o Customer, si existen.

---

## Roles

Un `User` puede tener uno o varios roles.

```text
User
 ├── ADMIN
 ├── SELLER
 └── CUSTOMER
```

La relación conceptual es:

```text
User 1 ─────── N UserRole N ─────── 1 Role
```

Los únicos roles son `ADMIN`, `SELLER` y `CUSTOMER`. La relación se mantiene separada para evitar acoplar la entidad `User` directamente a un único rol.

---

## Reglas de negocio relacionadas

* Un usuario debe tener un identificador único.
* El correo electrónico debe ser único dentro del sistema.
* Un usuario `INACTIVE` no puede acceder normalmente.
* Las credenciales nunca deben almacenarse en texto plano.
* Los cambios importantes relacionados con acceso deben poder auditarse.
* El usuario no debe eliminarse físicamente si existen registros históricos asociados.
* La desactivación debe realizarse mediante cambio de estado.

---

## Relaciones

### User → Role

Un usuario puede tener uno o varios roles.

```text
User 1 ─────── N UserRole N ─────── 1 Role
```

### User → Seller

Un usuario que posee el rol `SELLER` puede tener asociado un perfil de vendedor.

```text
User 1 ─────── 0..1 Seller
```

### User → Customer

Un usuario que posee el rol `CUSTOMER` puede tener asociado un perfil de cliente.

Todo perfil Seller exige que su User tenga el rol `SELLER`, y todo Customer exige `CUSTOMER`. La alta del rol y del perfil se confirma en la misma transacción; controles de integridad impiden retirar el rol mientras el perfil exista. Un User puede conservar varios roles, incluido `ADMIN`.

```text
User 1 ─────── 0..1 Customer
```

### User → Admin

No necesitamos inicialmente una entidad `Admin`.

El administrador puede representarse mediante:

```text
User + Role(ADMIN)
```

Esto evita crear una entidad innecesaria únicamente para representar permisos.

---

## Consideraciones de seguridad

La credencial persistida es `passwordHash`; la implementación de identidad fija el algoritmo y sus parámetros. Nunca se devuelve el hash ni se almacena la contraseña en claro. El MVP contempla inicio de sesión (CU-001) y no incorpora recuperación de contraseña, MFA, sesiones persistidas ni refresh tokens. Los tokens de acceso, si se usan, no son credenciales persistidas por el modelo MVP.

---

## Identificador

Se recomienda utilizar un identificador técnico independiente de datos modificables como correo electrónico.

Conceptualmente:

```text
UserId
```

El tipo concreto del identificador se definirá posteriormente durante el diseño técnico.

---

## Auditoría

Las operaciones relevantes sobre usuarios deben poder rastrearse.

Ejemplos:

* creación
* cambio de estado
* cambio de correo
* cambio de credenciales
* asignación o modificación de roles

Los eventos se registran en `AuditEvent`, sin guardar contraseñas, hashes, tokens, secretos ni valores sensibles. Las lecturas de credenciales de streaming también quedan auditadas.

---

## Fuera del alcance de User

No pertenece a `User`:

* saldo de billetera
* movimientos financieros
* inventario
* cuentas de streaming
* perfiles de streaming
* productos
* precios
* órdenes
* suscripciones
* pagos

Estos elementos pertenecen a otros módulos del dominio.

---

## Ejemplos

Un usuario con `Role(SELLER)` puede tener un perfil [[Seller]] y su billetera asociada. Un administrador se representa como `User + Role(ADMIN)`, sin una entidad `Admin` separada.

---

## Estado del diseño

**Definido.** `User.name` es la fuente vigente del nombre; `email` es único sin distinguir mayúsculas y `phone` es opcional. Inactivar o reactivar User no elimina ni modifica Seller, Customer, Order, WalletTransaction ni otros registros históricos. Los snapshots de Order se inicializan al crearla, se refrescan al confirmar y quedan congelados desde entonces.

Pendiente de implementación técnica:

* tipo de `UserId`
* mecanismo de autenticación
* hashing de contraseña
* persistencia
* índices
* restricciones SQL
