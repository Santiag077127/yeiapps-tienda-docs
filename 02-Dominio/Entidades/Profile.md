# Profile

## Descripción

`Profile` representa un perfil individual dentro de una [[Account]]. Es la unidad de inventario que puede asignarse cuando una [[ProductOffer]] se vende en modalidad `PROFILE`.

## Responsabilidades

* Mantener su pertenencia a una cuenta.
* Permitir determinar su disponibilidad para asignación individual.
* Conservar el vínculo histórico con el servicio asignado cuando corresponda.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del perfil. |
| `accountId` | Cuenta a la que pertenece. |
| `isActive` | Habilitación administrativa persistida para nuevas asignaciones. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

## Estados

Estados operativos derivados (no persistidos):

```text
AVAILABLE
ASSIGNED
BLOCKED_BY_FULL_ACCOUNT
INACTIVE
```

### AVAILABLE

El perfil puede utilizarse para una nueva asignación o venta `PROFILE` si la Account también tiene credenciales comerciales completas.

### ASSIGNED

El perfil está asignado actualmente a un servicio y no puede asignarse a otro beneficiario.

### BLOCKED_BY_FULL_ACCOUNT

La Account padre tiene una asignación `FULL_ACCOUNT` vigente. No hay asignación propia de Profile, pero tampoco disponibilidad para venta individual.

### INACTIVE

Se deriva cuando `Profile.isActive` o `Account.isActive` es falso, o si a la Account le faltan credenciales comerciales requeridas.

`EXPIRED` no es un estado de `Profile`: el vencimiento pertenece a [[Subscription]]. Tampoco vuelve automáticamente a `AVAILABLE` al vencer su servicio.

Los valores de esta sección se calculan; no son estados persistidos. Una asignación de `FULL_ACCOUNT` hace que sus Profiles deriven a `BLOCKED_BY_FULL_ACCOUNT`.

## Relaciones

* Cada `Profile` pertenece a una [[Account]].
* La asociación de al menos un Profile es obligatoria para toda Account al confirmar la transacción. Al crear, eliminar o trasladar un Profile, la frontera PostgreSQL bloquea primero las Accounts afectadas por UUID y solo después modifica el Profile; el mínimo se revalida al COMMIT y no exige que el Profile esté `AVAILABLE`.
* `accountId` puede corregirse solo antes de que el Profile tenga historial de asignación. Después es inmutable; moverlo cambiaría el significado histórico y podría eludir la exclusión entre `FULL_ACCOUNT` y `PROFILE`.
* Una [[ResourceAssignmentHistory]] referencia el perfil cuando se usa para prestar un período de servicio `PROFILE`. Una nueva Subscription de renovación puede usar el mismo perfil conforme al [[Product]].
* El estado de un perfil contribuye a determinar si su [[Account]] está `AVAILABLE`, `PARTIALLY_ASSIGNED` o `FULLY_ASSIGNED`.

## Reglas de negocio relacionadas

* Un perfil disponible puede ser asignado mediante una venta.
* Un perfil ya asignado no puede volver a venderse ni asignarse a otro cliente.
* Los otros perfiles de una cuenta pueden continuar disponibles después de vender uno individualmente.
* No se puede vender un perfil si su cuenta fue vendida como `FULL_ACCOUNT`.
* El perfil no decide por sí mismo la renovabilidad: esta se configura en el [[Product]].
* La asignación nueva de un perfil consume disponibilidad; una renovación de cliente que conserva el mismo perfil no la consume de nuevo.
* Si el producto es renovable internamente, el negocio puede reacondicionar manualmente el perfil fuera de la plataforma tras terminar el período. La plataforma no lo devuelve automáticamente al stock; solo vuelve a estar disponible al cargarse/publicarse nuevamente.
* Si un perfil falla durante un servicio vigente, una garantía puede asignar otro perfil de la misma cuenta o de otra cuenta. El perfil defectuoso queda no disponible para revisión y no vuelve automáticamente a la venta.

## Ejemplos

Si el perfil 1 falla durante un servicio vigente, puede reemplazarse por el perfil 2 disponible de la misma cuenta o por un perfil elegible de otra cuenta. La fecha de finalización del servicio no cambia.

## Fuera del alcance

No contiene precios, saldo, datos de comprador ni credenciales de cuenta expuestas libremente.

## Estado del diseño

**Definido conceptualmente.** Se persiste `isActive`; `AVAILABLE`, `ASSIGNED`, `BLOCKED_BY_FULL_ACCOUNT` e `INACTIVE` se derivan de habilitación, asignaciones vigentes y Account padre. La credencial pertenece a Account. Vencimiento y reacondicionamiento no devuelven stock automáticamente.
