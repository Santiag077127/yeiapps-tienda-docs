# Payment

## Descripción

`Payment` representa conceptualmente el pago externo que se requerirá cuando un [[Customer]] compre directamente en una fase posterior. No reemplaza la [[Wallet]] de un vendedor.

## Responsabilidades

* Mantener la referencia del pago externo asociado a una [[Order]].
* Separar el dominio de pagos del proveedor concreto.
* Permitir vincular la confirmación de pago con la operación comercial cuando se defina el flujo.

## Atributos conceptuales

| Atributo | Descripción |
| --- | --- |
| `id` | Identificador técnico del pago. |
| `orderId` | Orden relacionada. |
| `amount` | Importe solicitado o confirmado. |
| `status` | Estado de pago; catálogo definitivo pendiente. |
| `providerReference` | Referencia externa, cuando exista. |
| `createdAt` | Momento de creación. |
| `updatedAt` | Momento de la última actualización. |

Proveedor, moneda, estados y confirmación se definirán en la fase futura de compra directa de Customer.

## Estados

No se han definido estados para pagos. No deben inferirse a partir de los estados candidatos de [[Order]].

## Relaciones

* Un `Payment` se vincula a una [[Order]].
* La compra directa es realizada por un [[Customer]] en una fase posterior.
* La integración se abstraerá mediante un puerto/interfaz conceptual como `PaymentService`; no se ha seleccionado proveedor.

## Reglas de negocio relacionadas

* CUSTOMER no tiene billetera ni saldo interno.
* El pago directo de CUSTOMER debe utilizar un gateway externo en una fase posterior.
* No se debe hardcodear un proveedor concreto.
* Los registros históricos importantes deben preservarse.

## Ejemplos

Una compra directa futura de un customer puede crear una orden y un pago asociado; el proveedor que procesa ese pago aún no está elegido.

## Fuera del alcance

No debita la [[Wallet]] de SELLER, no es una implementación de gateway ni selecciona Mercado Pago, Wompi, PayU, Stripe u otro proveedor.

## Estado del diseño

**FASE FUTURA, fuera del modelo MVP.** Al abrir CU-025 se definirán proveedor, ciclo de vida, confirmación, reembolsos y datos; este documento no justifica una tabla Payment en la primera migración.
