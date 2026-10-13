# Account

## Descripción

`Account` representa un recurso real de inventario perteneciente a una [[Platform]]. Puede contener uno o varios [[Profile]] y participar en múltiples períodos de servicio a lo largo del tiempo; no equivale a una venta ni a una [[Subscription]].

## Responsabilidades

* Mantener la pertenencia a una plataforma y sus perfiles.
* Participar en la asignación de inventario para las modalidades `PROFILE` y `FULL_ACCOUNT`.
* Permitir derivar disponibilidad para nuevas asignaciones y conservar el control operativo de las cuentas asignadas.
* Conservar la información histórica aun después de una venta completa.
* Permitir identificar la cuenta como defectuosa/no disponible cuando falle durante una garantía, sin eliminarla del historial.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la cuenta. |
| `platformId` | Plataforma a la que pertenece. |
| `loginIdentifier` | Identificador operativo de acceso de la cuenta de streaming. |
| `secretPayload` | Credencial de streaming cifrada antes de persistirse. |
| `credentialMetadata` | Metadatos operativos no secretos de la credencial. |
| `isActive` | Habilitación administrativa persistida para nuevas asignaciones. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

`secretPayload` puede estar ausente al crear una Account administrativa o importada y no contiene texto plano: se cifra antes de persistirse. La clave de cifrado no vive en PostgreSQL y se administra mediante configuración segura o un secret manager. `User.passwordHash` es distinto: corresponde a autenticación del sistema y se almacena mediante hash seguro.

Sin identificador de acceso, `secretPayload` cifrado y metadatos de credencial requeridos y válidos, la Account no puede asignarse comercialmente. Es una condición de elegibilidad, no un nuevo estado.

## Estados

Estados operativos derivados (no persistidos):

```text
AVAILABLE
PARTIALLY_ASSIGNED
FULLY_ASSIGNED
INACTIVE
```

### AVAILABLE

La cuenta está habilitada, tiene las credenciales comerciales completas y no tiene asignación completa vigente. Para `FULL_ACCOUNT` no puede tener Profile previamente asignado; Profiles individuales pueden seguir disponibles.

### PARTIALLY_ASSIGNED

Uno o más perfiles están asignados y todavía existen perfiles disponibles. La cuenta ya no es elegible para `FULL_ACCOUNT`.

### FULLY_ASSIGNED

No existe disponibilidad para nuevas asignaciones, porque todos los perfiles están asignados o porque se asignó la cuenta completa. En este último caso los perfiles quedan no disponibles para nuevas ventas por la condición derivada `Account = FULLY_ASSIGNED`.

Esa condición no crea un estado persistido en [[Profile]]. La vista deriva `BLOCKED_BY_FULL_ACCOUNT` para explicar la indisponibilidad; `AVAILABLE`, `ASSIGNED` e `INACTIVE` también son valores de disponibilidad derivados.

### INACTIVE

`INACTIVE` se deriva si `isActive = false` o faltan credenciales comerciales completas. `AVAILABLE`, `PARTIALLY_ASSIGNED` y `FULLY_ASSIGNED` se derivan de perfiles habilitados y asignaciones vigentes. La proyección evita duplicar un estado que pueda contradecir el historial.

`EXPIRED` no es un estado de `Account`: el vencimiento pertenece a [[Subscription]]. Una cuenta defectuosa se marca `INACTIVE` para revisión y no queda disponible para nuevas asignaciones.

## Relaciones

* Cada `Account` pertenece a una [[Platform]].
* Una `Account` contiene uno o varios [[Profile]]. La Account y su primer Profile se crean atómicamente: si no se crean los Profiles, se revierte la Account. Ninguna Account puede confirmarse al COMMIT sin Profile; una Account administrativa incompleta solo puede tener credenciales pendientes, no cero Profiles.
* No se elimina ni desvincula el último Profile requerido de una Account comercializable. La operación entra por la frontera de escritura de inventario, bloquea primero la Account raíz y luego revalida Profiles y asignaciones; si deja una configuración no permitida, se revierte.
* `platformId` puede corregirse antes de que la Account o cualquiera de sus Profiles tenga historial de asignación. Tras el primer uso comercial es inmutable, para no reinterpretar asignaciones históricas.
* Una [[ResourceAssignmentHistory]] referencia la cuenta cuando se usa para prestar un período de servicio `FULL_ACCOUNT`.
* El estado de cuenta se deriva de la disponibilidad de sus perfiles o de una asignación completa, sin reemplazar el estado del servicio en [[Subscription]].

## Reglas de negocio relacionadas

* Una cuenta completa solo se puede vender si ninguno de sus perfiles se asignó o vendió previamente.
* Tras venderse como `FULL_ACCOUNT`, deja de estar disponible para nuevas ventas y sus perfiles quedan bloqueados/no disponibles.
* La cuenta vendida no se elimina ni se retira del historial.
* No se puede vender un perfil de una cuenta ya vendida completamente.
* Una nueva asignación de cuenta consume una unidad de disponibilidad para nuevas asignaciones.
* Solo una cuenta `AVAILABLE` puede asignarse como `FULL_ACCOUNT`.
* Una cuenta `PARTIALLY_ASSIGNED` puede seguir aportando perfiles disponibles, pero no puede venderse completa.
* La cuenta no determina por sí misma si un servicio es renovable; esa regla se configura explícitamente en el [[Product]].
* Si el producto permite renovación para cliente y se conserva esta misma cuenta, no se consume una unidad adicional de stock.
* Si una cuenta falla durante un servicio vigente, una garantía finaliza su asignación histórica y registra otra cuenta como reemplazo, sin crear una nueva compra ni extender el período.
* Una cuenta defectuosa no queda disponible automáticamente para nuevas ventas; requiere revisión/diagnóstico y posterior reincorporación administrativa.
* El vencimiento de una suscripción no devuelve automáticamente la cuenta al stock ni cambia sus credenciales, sesiones o perfiles. El reacondicionamiento es manual y externo a la plataforma; solo una nueva carga/publicación la hace disponible otra vez.

## Ejemplos

La cuenta `Cuenta 001` puede participar en un período de servicio de 30 días y, si el producto permite renovación para cliente, en un nuevo período usando el mismo recurso. Si falla el día 13, una cuenta de reemplazo conserva la fecha de finalización original. Si se vende completa, sus cuatro perfiles dejan de poder venderse individualmente.

## Fuera del alcance

No define precios, el comprador ni pagos. Las credenciales solo se exponen en operaciones autorizadas y cada lectura queda auditada desde el MVP.

## Estado del diseño

**Definido conceptualmente.** `isActive` se persiste como habilitación manual; disponibilidad y estado se derivan de asignaciones/Profiles mediante vistas. No puede inactivarse con asignación vigente. Credenciales necesarias para comercializar se cifran antes de guardarse; la clave vive fuera de PostgreSQL. Toda lectura de credenciales se audita. La rotación no modifica snapshots históricos de Order.
