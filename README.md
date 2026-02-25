# Laundry Ops — Arquitectura y Diseño

Sistema de gestión para lavanderías diseñado para reemplazar el registro manual en papel por una plataforma digital accesible desde cualquier dispositivo.

---

## 1. Arquitectura del Sistema (Modelo C4)

### Diagrama de Contexto
Muestra a los actores principales y los sistemas externos con los que interactúa la plataforma.
![Contexto C4](./technical/c4-context.png)

### Diagrama de Contenedores
Zoom a la arquitectura interna: Frontend (SPA), Backend (API REST) y Base de Datos.
![Contenedores C4](./technical/c4-container.png)

---

## 2. Decisiones de Ingeniería y Diseño

El valor de este proyecto radica en las decisiones técnicas orientadas a resolver problemas del negocio. La justificación detallada se encuentra en los siguientes documentos (ADR):

* 📄 **[Stack Tecnológico y Arquitectura (stack.md)](./technical/stack.md)**: Justificación del monolito modular, máquina de estados con `django-fsm` y transacciones atómicas.
* 📄 **[Diseño de Base de Datos (database-design.md)](./technical/database-design.md)**: Reglas de negocio aplicadas al modelo relacional (inmutabilidad financiera, auditoría operativa y separación de contextos).
* 📊 **[Diagrama Entidad-Relación (ERD)](./technical/erd.png)**
* 🎯 **[User Stories (user-stories.md)](./product/user-stories.md)**: Casos de uso priorizados y definidos usando convención Gherkin.

---

## 3. Descripción del Problema y Solución

**El Problema:** Actualmente la lavandería opera con tickets en papel autocopiativo, generando:
- Nula visibilidad en tiempo real de las órdenes pendientes, en proceso o listas.
- Control de caja opaco (dificultad para rastrear pagos parciales, saldos y Yape/Plin).
- Trazabilidad inexistente para prendas enviadas a terceros (lavado al seco).

**La Solución:** Una plataforma web responsive que permite:
- Registrar órdenes y gestionar pagos parciales con cálculo automático de saldos.
- Controlar el flujo de estados (`Recibido` → `En proceso` → `Listo` → `Entregado`).
- Visualizar ingresos diarios y métricas operativas desde un dashboard.
---

## 4. Ecosistema de Repositorios

El sistema está dividido para simular un entorno de despliegue real:

| Repositorio | Descripción |
|---|---|
| [laundry-ops-api](https://github.com/laundry-erp/laundry-ops-api) | Backend API REST (Django + DRF) |
| [laundry-ops-web](https://github.com/laundry-erp/laundry-ops-web) | Frontend UI (React + Tailwind CSS) |
| **[laundry-ops-architecture](https://github.com/bry4nbe/laundry-ops-architecture)** | **Documentación y diseño (Este repositorio)** |

---

## Estructura de este repositorio

```
laundry-ops-architecture/
├── README.md
├── product/
│   ├── problem-and-solution.md
│   └── user-stories.md
├── technical/
│   ├── stack-decisions.md
│   ├── database-design.md
│   ├── erd.png
│   ├── c4-context.png
│   └── c4-container.png
└── infrastructure/
    ├── deployment.md
    └── ci-cd.md
```

---

## Autor

Desarrollado por **Bryan Barba**.  
Stack: Django · React · PostgreSQL · Tailwind CSS