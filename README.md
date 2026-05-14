# 🛠️ Documentación de Arquitectura: StockFlow (SaaS Inventory)

**Estado:** Fase de Diseño Inicial  
**Arquitectura:** N-Tier / Multi-tenancy por aislamiento lógico  
**Stack:** Nuxt 3, NestJS, Prisma, PostgreSQL

---

## 1. Arquitectura de Sistema (Vista de Componentes)

El sistema se rige por un desacoplamiento total entre capas para permitir el escalado independiente de cada una.

*   **Frontend (Nuxt 3):** Cliente SPA/SSR que gestiona el estado de sesión y el contexto del `activeProjectId`.
*   **Backend (NestJS + Prisma):** API REST modular. Cada recurso es un módulo independiente que se comunica a través de servicios inyectados.
*   **Database (PostgreSQL):** Motor relacional encargado de la integridad referencial y transaccionalidad (ACID).

---

## 2. Modelo de Datos (ERD)

Se opta por **UUID v4** como llave primaria para todos los registros. Esto previene ataques de enumeración (seguridad) y facilita el escalado horizontal de la base de datos.

### Especificaciones de Relación
*   **User ↔ Project (1:N):** Propiedad directa del espacio de trabajo.
*   **Project ↔ Category (1:N):** Las categorías son *scoped* al proyecto (no globales).
*   **Project ↔ Product (1:N):** Relación de propiedad fuerte para asegurar aislamiento de inventario.
*   **Product ↔ Category (N:1):** Clasificación requerida para lógica de negocio.
*   **Project ↔ Gallery (1:N):** Repositorio centralizado de assets por proyecto.

---

## 3. Estrategia de Multi-tenancy y Seguridad

Para garantizar que los datos de un proyecto sean inaccesibles para usuarios ajenos, implementamos **Aislamiento Lógico**.

| Nivel | Estrategia de Escalabilidad | Seguridad (Security+ Focus) |
| :--- | :--- | :--- |
| **Identidad** | JWT Stateless (RS256) | Evita almacenamiento de sesiones; protección contra secuestro. |
| **Acceso** | Guards/Middleware de NestJS | Prevención de **BOLA** (Broken Object Level Authorization). |
| **Datos** | Partitioning Lógico (`project_id`) | Todas las queries incluyen forzosamente `WHERE project_id = $ID`. |

---

## 4. Gestión de Archivos (Referencia de Objetos)

Evitamos el almacenamiento de BLOBs en la base de datos para mantener el performance de los backups.

1.  **Ingesta:** Procesamiento de archivos mediante bus de datos en NestJS (Multer).
2.  **Storage:** Almacenamiento físico organizado por `projectId`: `/uploads/{project_id}/{hash}.webp`.
3.  **Persistencia:** Tabla `Gallery` almacena metadatos y la URL pública del asset.
4.  **Asociación:** La tabla `Product` referencia los UUIDs de la galería para evitar duplicidad de archivos.

---

## 5. Justificación del Stack Tecnológico

### **Node.js (NestJS)**
*   **Escalabilidad:** Manejo asíncrono de peticiones (Event Loop), ideal para operaciones de inventario concurrentes.
*   **Estandarización:** TypeScript garantiza consistencia de tipos desde la DB hasta el Front.

### **Prisma ORM**
*   **Type-Safety:** Generación automática de tipos basada en el esquema de base de datos.
*   **Mantenimiento:** Sistema de migraciones declarativo que asegura paridad entre entornos.

### **PostgreSQL**
*   **Integridad:** Aplicación de reglas `ON DELETE CASCADE` para limpieza automatizada de datos.
*   **Flexibilidad:** Soporte nativo para `JSONB`, permitiendo atributos dinámicos en productos sin alterar el esquema.

---

> **Nota de Seguridad:** Se implementará **Rate Limiting** y **Hashing de Contraseñas (Argon2)** para mitigar ataques de fuerza bruta y cumplir con estándares de ciberseguridad industrial.