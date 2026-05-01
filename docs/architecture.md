# Arquitectura del Sistema (Método BMAD) - Estándar Universal

## 1. Capa de Intención (Intent)
*Define el propósito y el flujo de comunicación de alto nivel.*
- **Actor:** Usuario (Product Owner) + IA (Arquitecto Senior).
- **Objetivo:** Traducir ideas o código existente en especificaciones técnicas claras.
- **Interfaz:** Cursor operando bajo la rule `.cursor/rules/architect-brain.mdc` (cargada automáticamente con `alwaysApply: true`).
- **Cerebro:** `@prompts/architect-brain.md` (Orquestador de lógica y calidad).

## 2. Capa de Implementación (Implementation)
*Define las reglas de construcción y la ubicación de la verdad.*
- **Single Source of Truth (SSOT):** Carpeta `docs/` en la raíz. Ninguna documentación vive fuera de aquí.
- **Flujo de Trabajo:** 1. Identificar necesidad/error.
    2. Sincronizar en `user-stories.md` y `roadmap.md` (Estado: [SYNC_PENDING]).
    3. Ejecutar código basándose en `blueprints.md`.
- **Estándares Técnicos (Blueprints):**
    - Referencia obligatoria para: Conexiones DB (PDO), Patrones UI (AJAX/Modales), y Librerías (JS/PHP).

## 3. Capa de Memoria (Skills & Knowledge)
*Define cómo la IA aprende reglas específicas sin ensuciar la arquitectura global.*
- **Dynamic Skills:** La IA genera archivos en `.cursor/skills/<nombre>/SKILL.md` para lógicas complejas (ej. Reglas fiscales, APIs externas, procesos SFTP).
- **Persistencia:** Estas skills permiten que la IA "recuerde" cómo programar módulos específicos en el futuro.

## 4. Ciclo de Vida del Sistema (Spec-kit Lifecycle)
*Estados por los que pasa cualquier tarea o inicio de proyecto.*



- **[SCANNING]:** Análisis inicial de la raíz para detectar modo (Nuevo vs Existente).
- **[SYNC_PENDING]:** Pausa obligatoria de seguridad para actualizar documentos antes de programar.
- **[ARCHITECTING]:** Redacción de requisitos en `docs/`.
- **[CODING]:** Construcción técnica siguiendo Blueprints y Skills.
- **[DONE]:** Ticket cerrado, código validado y documentación al día.