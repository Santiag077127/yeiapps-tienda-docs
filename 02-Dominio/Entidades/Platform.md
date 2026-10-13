# Platform

## Descripción

`Platform` representa un servicio de streaming que la plataforma comercializa y cuyo inventario se administra mediante [[Account]]. No representa una cuenta, un perfil ni un precio.

## Responsabilidades

* Identificar el servicio de streaming al que pertenecen las cuentas.
* Agrupar las cuentas de inventario asociadas.
* Permitir que el catálogo relacione productos comerciales con la plataforma correspondiente.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la plataforma. |
| `name` | Nombre comercial de la plataforma. |
| `status` | Estado operativo: `ACTIVE` o `INACTIVE`. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

## Estados

| Estado | Significado |
| --- | --- |
| `ACTIVE` | Puede participar en nuevas ventas y asignaciones si las demás entidades son elegibles. Estado inicial. |
| `INACTIVE` | Impide nuevas ventas y asignaciones de sus recursos, sin afectar historial ni servicios vigentes. |

Transiciones permitidas: `ACTIVE → INACTIVE` y `INACTIVE → ACTIVE`. No hay cascadas de estado hacia Product, Account o Profile. Reactivar solo vuelve a habilitar la elegibilidad futura de recursos que ya cumplan las demás reglas.

Toda transición distinta de las anteriores está prohibida. La inactivación no cancela Subscription vigentes ni modifica sus fechas.

## Relaciones

* Una `Platform` agrupa cero o más [[Account]].
* Uno o más [[Product]] pertenecen a esta plataforma.

## Reglas de negocio relacionadas

* Una `Platform` representa un servicio de streaming comercializado.
* La gestión de plataformas corresponde al ADMIN.
* Desactivar una plataforma no debe borrar las operaciones históricas vinculadas.

## Ejemplos

Una plataforma puede ser “Netflix”; sus cuentas de inventario se documentan como [[Account]], no como atributos de la plataforma.

## Fuera del alcance

No contiene credenciales, perfiles, precios, órdenes ni suscripciones.

## Estado del diseño

**Definido conceptualmente.** La inactivación preserva todos los registros históricos y no cancela Subscription vigentes.
