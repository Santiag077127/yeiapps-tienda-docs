# Streaming Business System — documentación

Documentación funcional y de datos para la tienda de servicios de streaming.

## Estado

🟢 Especificación MVP lista para iniciar el modelo lógico. No se ha creado DDL ni migraciones de negocio. La moneda, zona horaria y contrato de credenciales son configuración requerida por despliegue; no bloquean el esquema.

## Inicio

- [[AUDITORIA-FINAL-PRE-BD]] — veredicto, alcance, decisiones cerradas, integridad y siguiente paso.
- [[Vision]] — propósito y límites.
- [[Alcance-MVP]] — casos incluidos/futuros, sin HUs inventadas.
- [[Roadmap]] — etapas de documentación e implementación.

## Negocio y dominio

- [[Actores]], [[Casos-de-Uso]] y [[Reglas-de-Negocio]].
- [[Mapa-del-Dominio]] y `02-Dominio/Entidades/`.
- [[Compra-Seller]], [[Venta-Account]], [[Recarga-Wallet]] y [[Compra-Customer]].

## Arquitectura y datos

- [[Arquitectura-General]] y [[Modulos]].
- [[Modelo-ER]], [[Relaciones]], [[Modelo-Fisico-PostgreSQL]] y [[Trazabilidad-CU-BD]].
- El plan documental de changesets y estado de Liquibase está en `../yeiapp-store-db/README.md`.

## Siguiente paso

Crear/revisar el modelo lógico definitivo y después generar DDL PostgreSQL y migraciones Liquibase conforme al plan de dependencias. [[Decisiones-Pendientes]] documenta criterios cerrados y condiciones fuera del modelo. La matriz conecta CU → RN → entidad/atributos → operación → persistencia objetivo.
