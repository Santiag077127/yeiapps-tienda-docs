# WalletTransaction

## Descripción

`WalletTransaction` es un movimiento financiero inmutable de una [[Wallet]]. Permite reconstruir las recargas y débitos que explican el saldo del vendedor.

## Responsabilidades

* Registrar el efecto financiero de una operación.
* Conservar saldo anterior y saldo posterior.
* Mantener la referencia y el actor que originaron el movimiento para auditoría.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del movimiento. |
| `walletId` | Billetera afectada. |
| `sequenceNo` | Número contable monotónico y único dentro de la Wallet. |
| `orderId` | Obligatorio únicamente para `PURCHASE_DEBIT`; nulo para los demás tipos. |
| `reversedWalletTransactionId` | Movimiento original revertido; obligatorio únicamente para `REVERSAL`. |
| `type` | `TOP_UP`, `PURCHASE_DEBIT`, `ADJUSTMENT_CREDIT`, `ADJUSTMENT_DEBIT` o `REVERSAL`. |
| `amount` | Delta firmado del movimiento. |
| `previousBalance` | Saldo antes del movimiento. |
| `resultingBalance` | Saldo después del movimiento. |
| `currencyCode` | Moneda ISO 4217 de la Wallet y del movimiento. |
| `reference` | Referencia a la operación originadora. |
| `originatedBy` | Usuario que originó la operación. |
| `occurredAt` | Momento del movimiento. |
| `displayStatus` | Estado derivado para CU-008: `REVERSAL` para la fila de reverso; para originales, `REVERSED` si existe reverso que lo referencia o `NOT_REVERSED`. No es columna persistida. |

## Tipos y reglas financieras

Los movimientos son inmutables. `TOP_UP` y `ADJUSTMENT_CREDIT` tienen `amount > 0`; `PURCHASE_DEBIT` y `ADJUSTMENT_DEBIT`, `amount < 0`. `REVERSAL` es un nuevo movimiento que revierte uno anterior sin modificarlo ni eliminarlo; su signo corresponde al efecto de la reversión.

Un original admite como máximo un reverso total. Solo son reversibles `TOP_UP`, `PURCHASE_DEBIT`, `ADJUSTMENT_CREDIT` y `ADJUSTMENT_DEBIT`; un `REVERSAL` no se revierte. El reverso debe pertenecer a la misma Wallet, conservar moneda y tener importe exactamente opuesto. Si se requiere una corrección posterior, se registra un nuevo ajuste autorizado, nunca se altera un movimiento previo.

Una Order admite como máximo un `PURCHASE_DEBIT`; la restricción física parcial sobre `orderId` evita doble cobro por reintentos concurrentes. Al crear un movimiento, `previousBalance` es el saldo bloqueado inmediatamente anterior y `resultingBalance` el saldo inmediatamente posterior. Wallet mantiene ese último resultado y no puede actualizarse por fuera de la transacción que inserta el movimiento.

`sequenceNo` es asignado exclusivamente por el ledger mientras la Wallet está bloqueada. Es estrictamente creciente por Wallet, se aplica a todos los tipos de movimiento y determina el orden contable; `occurredAt` registra el instante del evento, pero no sustituye esa secuencia. El movimiento con mayor `sequenceNo` es el que explica el saldo materializado actual.

Siempre se cumple `resultingBalance = previousBalance + amount` y `resultingBalance >= 0`.

## Relaciones

* Cada movimiento pertenece a una [[Wallet]].
* `PURCHASE_DEBIT` referencia obligatoriamente una [[Order]]; los demás tipos no la referencian de forma directa. Un reverso conserva el vínculo con su original, del cual se deriva la orden si existe.
* Una recarga o ajuste administrativo registra al ADMIN que la originó.
* Un actor de compra debe estar autorizado para el Seller buyer; un reverso requiere actor autorizado. La FK del actor prueba existencia, pero la autorización, rol y estado `ACTIVE` se revalidan dentro de la transacción.

## Reglas de negocio relacionadas

* Toda recarga genera un registro histórico.
* Una compra de SELLER debe registrar el movimiento de billetera correspondiente.
* Los movimientos financieros son inmutables y no deben eliminarse físicamente.
* El saldo resultante no puede ser negativo.
* Un `PURCHASE_DEBIT` referencia la [[Order]] correspondiente.
* CU-008 presenta el tipo y el estado derivado de reverso. El movimiento no tiene ciclo de estado propio.

## Ejemplos

Una recarga usa `TOP_UP` positivo y una compra usa `PURCHASE_DEBIT` negativo. Un reversal conserva la transacción original y agrega un movimiento contrario trazable.

## Fuera del alcance

No es una orden, una suscripción ni una pasarela de pago; tampoco modifica retrospectivamente el precio histórico de un ítem de orden.

## Estado del diseño

**Definido conceptualmente.** No existe entidad `Refund` en el MVP; los reversos se representan mediante nuevos movimientos `REVERSAL`.
