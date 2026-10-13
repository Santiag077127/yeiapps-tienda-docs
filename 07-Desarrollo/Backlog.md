# Backlog documental

No hay Historias de Usuario formalizadas en el repositorio. No se inventan HUs para completar esta auditoría; hasta que el Product Owner publique el backlog, los Casos de Uso CU-001–CU-036 son la referencia funcional y se clasifican en [[Alcance-MVP]].

## Secuencia de trabajo propuesta

1. Revisar las decisiones explícitas de [[Decisiones-Pendientes]] con negocio.
2. Validar [[Modelo-ER]] y matriz [[Trazabilidad-CU-BD]].
3. Congelar [[Modelo-Fisico-PostgreSQL]], incluida estrategia de vistas y funciones de escritura.
4. Crear changesets pequeños en `yeiapp-store-db`, por dependencia, con permisos mínimos.
5. Implementar servicio de aplicación y frontend contra las reglas de atomicidad y autorización.
6. Agregar pruebas cuando se inicie la fase de implementación y el equipo solicite su definición.

## Estado actual

Especificación previa a BD en revisión. Los changelogs Liquibase son scaffolds vacíos; no hay migraciones listas para aplicar ni backlog técnico estimado.
