# AuditEvent

## Propósito

`AuditEvent` registra quién ejecutó una acción sensible, cuándo ocurrió, sobre qué objeto y con qué contexto seguro. Es transversal; no reemplaza los registros financieros de [[WalletTransaction]] ni el historial de [[ResourceAssignmentHistory]].

## Atributos conceptuales

| Atributo | Descripción |
|---|---|
| `id` | Identificador del evento. |
| `actorType` | `USER` o `SYSTEM`. |
| `actorUserId` | User responsable para actor `USER`; nulo para `SYSTEM`. |
| `actionCode` | Operación concreta. |
| `entityType`, `entityId` | Tipo e identificador del objeto afectado; referencia polimórfica. |
| `occurredAt` | Momento UTC. |
| `changedFields` | Before/after de valores no secretos, cuando aplique. |
| `details` | Razón/resultado/contexto estructurado no sensible. |
| `requestId` | Correlación opcional con la solicitud. |

## Reglas

- Append-only: la aplicación no modifica ni elimina eventos.
- Audita cambios en User/roles, catálogo/precios, credenciales, relaciones SellerCustomer, asignaciones/liberaciones, reposición, renovación y cancelación.
- Cada lectura de credencial de streaming genera un evento con actor, Account, momento, ámbito de servicio y resultado; nunca incluye el valor leído.
- No contiene password, password hash, token, secreto, ciphertext ni datos sensibles innecesarios. `changedFields` usa allowlist y omite campos confidenciales.
- User actor debe existir; `SYSTEM` representa job/operación interna sin identidad humana.
- Si una acción y su auditoría deben ser atómicas, el evento se inserta en la misma transacción. Para lecturas de secreto, auditar de forma que el acceso solo se entrega si la auditoría requerida pudo persistirse.

## Persistencia objetivo

`identity.audit_event`; estructura detallada en [[Modelo-Fisico-PostgreSQL]]. La relación polimórfica con objetos del dominio no es una FK física para permitir auditoría de múltiples esquemas y eventos retenidos. Las vistas y retención no deben permitir modificación del histórico.
