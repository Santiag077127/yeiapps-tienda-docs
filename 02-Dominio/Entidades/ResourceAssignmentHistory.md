# ResourceAssignmentHistory

## Descripción

`ResourceAssignmentHistory` asocia una [[Subscription]] con cada Account/Profile utilizado durante su período. Registra asignación inicial, reposiciones y liberación. No es entidad de Incident, Guarantee, Replacement, Renewal ni Reconditioning.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `subscriptionId` | Período atendido. |
| `accountId` | Account asignada para `FULL_ACCOUNT`, cuando corresponda. |
| `profileId` | Profile asignado para `PROFILE`, cuando corresponda. |
| `assignedAt` | Momento UTC en que empieza la asignación. |
| `releasedAt` | Momento UTC en que se libera; nulo mientras el recurso siga retenido por el servicio. |
| `reason` | Motivo de asignación: `INITIAL_ASSIGNMENT` o `WARRANTY_REPLACEMENT`. |
| `sequence` | Orden único dentro de la Subscription. |
| `performedByUserId` | Usuario que realizó la asignación/reposición. |
| `failureOccurredAt` | Momento reportado de falla, obligatorio para `WARRANTY_REPLACEMENT`. |
| `failureDescription` | Descripción breve no sensible de la falla, obligatoria para garantía. |
| `releasedByUserId` | Usuario que realizó la liberación; nulo hasta liberar. |
| `releaseReason` | `WARRANTY_REPLACEMENT`, `RENEWAL_TRANSFER` o `MANUAL_RECONDITIONING`; nulo hasta liberar. |

Cada fila refiere exactamente una Account o Profile, según modalidad, nunca ambas. `releasedAt`, `releasedByUserId` y `releaseReason` se completan juntos una única vez. `releasedAt` debe ser posterior a `assignedAt`.

## Historial de falla y reposición

`INITIAL_ASSIGNMENT` usa secuencia 1 y no tiene datos de falla. `WARRANTY_REPLACEMENT` usa secuencia posterior y exige falla, actor y asignación previa liberada. La fila anterior identifica el recurso que falló; la nueva fila identifica el recurso sustituto y conserva la fecha de vencimiento de la misma Subscription. `failureOccurredAt` y `failureDescription` constituyen el mínimo de incidencia del MVP; AuditEvent conserva además el contexto seguro. No se requiere una entidad Incident.

## Liberación y disponibilidad

Una asignación está retenida mientras `releasedAt` sea nulo, incluso si la Subscription ya venció. El vencimiento no libera el recurso ni lo devuelve al stock. La liberación ocurre únicamente como parte de una reposición (`WARRANTY_REPLACEMENT`), de una transferencia de recurso a un período renovado (`RENEWAL_TRANSFER`) o de la publicación administrativa tras reacondicionamiento externo (`MANUAL_RECONDITIONING`). El evento guarda el actor y la razón.

Al renovar reutilizando el mismo recurso, la transacción cierra la fila del período anterior y crea la `INITIAL_ASSIGNMENT` de la nueva Subscription de forma atómica, con el mismo recurso y sin consumir stock adicional. Si se utiliza otro recurso, la asignación anterior permanece retenida hasta que el administrador la libere al publicar el recurso reacondicionado.

No tiene un estado separado: una asignación está retenida si no se ha completado su liberación. La disponibilidad de Account/Profile también considera habilitación, credenciales, padre y modalidad; no se reduce a esta columna.

## Relaciones e integridad

* Cada Subscription persistida tiene exactamente una asignación inicial válida y cero o más reposiciones.
* Cada fila usa exactamente una Account o Profile.
* Una Subscription y un Profile pueden tener como máximo una asignación retenida; una Account puede tener como máximo una asignación retenida `FULL_ACCOUNT` y múltiples Profile distintas.
* Exclusiones temporales impiden solapamientos del mismo Profile o de una Account en modalidad `FULL_ACCOUNT`.
* Una asignación `FULL_ACCOUNT` excluye asignaciones de cualquiera de los Profiles de esa Account.
* El historial es append-only salvo completar una única vez los campos de liberación. El actor, motivo y momento de liberación quedan tanto en esta relación como en AuditEvent.

## Fuera del alcance

No crea una venta, no cobra, no inicia renovaciones ni decide el destino del recurso defectuoso. El reacondicionamiento material ocurre fuera de la plataforma.

## Estado del diseño

**Cerrado para el MVP.** Recurso, fechas, secuencia, actores, motivos de asignación y liberación, falla y reposición están especificados físicamente.