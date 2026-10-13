# Wallet

## Descripción

`Wallet` representa la billetera de un [[Seller]]. Mantiene su saldo y el historial inmutable de sus movimientos mediante [[WalletTransaction]]. Un [[Customer]] no posee billetera ni saldo interno.

## Responsabilidades

* Mantener el saldo disponible del vendedor.
* Registrar recargas administrativas y débitos por compras.
* Impedir que una compra deje el saldo por debajo de cero.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico de la billetera. |
| `sellerId` | Vendedor titular. |
| `balance` | Saldo disponible. |
| `currencyCode` | Moneda única de la Wallet, código ISO 4217 de tres caracteres. |
| `nextSequenceNo` | Próximo número contable, administrado exclusivamente por el ledger. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

Los importes usan precisión decimal de hasta cuatro decimales; el redondeo comercial final es a dos decimales. El MVP es monomoneda: cada instalación configura un código ISO 4217 (`store.currency_code`), copiado a cada Wallet y luego inmutable. No se presupone COP ni existe conversión automática. El operador fija el código como configuración antes de crear Wallets; el valor de despliegue no cambia el diseño.

## Estados

La Wallet no tiene estado propio en el MVP. La elegibilidad depende de que Seller/User estén activos y haya saldo suficiente.

## Relaciones

* Cada [[Seller]] tiene una única `Wallet`.
* Una `Wallet` contiene cero o más [[WalletTransaction]].
* Las compras del vendedor generan el débito correspondiente, vinculado a la [[Order]] que lo originó.

## Reglas de negocio relacionadas

* El ADMIN puede recargar la billetera de un vendedor.
* El vendedor no puede gastar más que su saldo disponible.
* Las operaciones financieras deben poder auditarse.
* Los movimientos financieros no se eliminan físicamente.
* La Wallet nace automáticamente con el Seller, saldo cero, `nextSequenceNo = 1` y sin movimiento ficticio. No puede moverse a otro Seller ni eliminarse mientras su Seller exista.

## Ejemplos

Una recarga incrementa el saldo y una compra confirmada lo reduce; ambos efectos se registran como movimientos separados.

## Fuera del alcance

No pertenece a CUSTOMER, no procesa gateways de pago ni almacena el catálogo, las cuentas o los perfiles.

## Estado del diseño

**Definido conceptualmente.** La confirmación de una compra coordina Wallet, Order, inventario, Subscription y [[WalletTransaction]] en una misma transacción. El saldo materializado cambia únicamente como parte de la inserción del movimiento financiero correspondiente.

La Wallet que financia la compra pertenece obligatoriamente al Seller buyer de la Order. El saldo es el resultado del movimiento con mayor `sequenceNo` —o cero si no hay movimientos— y los movimientos concurrentes se serializan bloqueando esa Wallet. `nextSequenceNo` se incrementa junto con el movimiento, saldo y ledger; no es modificable por operaciones administrativas.

La cardinalidad mínima se verifica al `COMMIT` de la transacción de alta mediante una restricción diferida: un Seller persistido tiene exactamente una Wallet. La unicidad de `sellerId` evita una segunda Wallet.
