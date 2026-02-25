# Stack Tecnológico — Laundry Ops

## Contexto del Proyecto

Sistema de gestión operativa para una lavandería de barrio en Lima, orientado a resolver tres problemas concretos del negocio:

1. **Pérdida de control sobre órdenes en proceso:** no hay visibilidad de qué prendas están recibidas, en lavado o listas para entrega.
2. **Flujo de caja opaco:** los pagos parciales (adelantos) se registran en papel y es difícil saber cuánto se cobró y cuánto falta por cobrar en el día.
3. **Trazabilidad nula del lavado al seco:** las prendas enviadas a terceros no tienen seguimiento formal, generando demoras y pérdidas.

El sistema reemplaza un proceso basado en tickets de papel autocopiativo (costo actual: S/540/año) por una plataforma web responsive, operable desde un celular Android en el mostrador y desde desktop en administración.

**Restricciones del proyecto:** un solo desarrollador, una sucursal, dos roles de usuario (administrador y operador), operadora principal con experiencia digital limitada.

---

## Estado Actual

| Componente | Estado |
|---|---|
| User stories y criterios de aceptación | ✅ Definidas |
| Decisiones de arquitectura | ✅ Documentadas |
| Modelo de datos (ERD) | ✅ Diseñado |
| Diagramas C4 | ✅ Contexto y contenedores |
| Backend (API REST) | 🔲 En desarrollo |
| Frontend (SPA) | 🔲 En desarrollo |
| Tests | 🔲 En desarrollo |
| CI/CD pipeline | 🔲 Pendiente |
| Deploy en producción | 🔲 Pendiente |

---

## Decisiones Arquitectónicas Transversales

### Monolito modular

El volumen del sistema (una sucursal, decenas de órdenes diarias) no justifica microservicios. La separación de responsabilidades se implementa a nivel de código mediante capas (`models`, `services`, `serializers`, `views`), no de infraestructura. Cada capa es independientemente testeable y el sistema es extensible sin rediseño estructural.

### Máquina de estados con consistencia entre entidades

El sistema gestiona dos máquinas de estado independientes que deben mantenerse coherentes:

- `Order.status`: `received → in_process → ready → delivered`
- `OrderItem.dry_cleaning_status`: `pending_send → sent → returned`

Una inconsistencia entre ambas (orden en `ready` con items aún en `sent`) es un bug con impacto operativo directo: la operadora le diría al cliente que su ropa está lista cuando en realidad sigue con el tercero.

La solución tiene dos capas:

**`django-fsm`** define las transiciones válidas directamente en el modelo, rechazando cualquier transición inválida desde cualquier punto de entrada, incluyendo el Django Admin. Se eligió sobre validaciones manuales en `services.py` porque las reglas viven en el modelo y son imposibles de saltarse desde cualquier capa, no solo desde la API.

**Transacciones atómicas** en `services.py` envuelven los cambios de estado de orden e items relacionados. Si la validación de consistencia falla, ningún cambio llega a la base de datos. Una signal `post_save` en `OrderItem` evalúa si todos los items de lavado al seco de la orden están en `returned` antes de permitir la transición de la orden a `ready`.

---

## Backend

### Python 3.12 + Django 5 + Django REST Framework

**Django Admin operativo.** El panel `/admin` resuelve el mantenimiento de datos en producción sin construir pantallas adicionales: corrección de registros, gestión de usuarios, consulta de estados. Esto elimina semanas de desarrollo de backoffice.

**Migraciones automáticas.** Los cambios en el modelo de datos se traducen en migraciones con `makemigrations`, sin SQL manual. Crítico para desarrollo iterativo donde el modelo evoluciona frecuentemente.

**Arquitectura por capas testeable.** La lógica de negocio vive en `services.py`, separada de vistas y serializadores. Cada función de servicio es testeable de forma aislada sin levantar el servidor HTTP.

*Alternativas descartadas:* FastAPI requiere ensamblar ORM, migraciones y autenticación por separado; costo que no se justifica para este contexto.

### Autenticación: JWT con djangorestframework-simplejwt

Frontend y backend operan en dominios distintos (Vercel y Railway), consecuencia directa de haber elegido una SPA con React (ver justificación en la sección de Frontend). En ese contexto, JWT es el mecanismo estándar: `refresh_token` en cookie HTTP-only, `access_token` en memoria con vida de 15 minutos. Los permisos por rol se validan mediante permission classes de DRF en cada endpoint.

*Nota:* Si el frontend fuera server-rendered con Django templates, Django sessions sería suficiente y más simple. JWT es la consecuencia natural de la arquitectura SPA elegida.

### PostgreSQL 16

`DECIMAL(10, 2)` garantiza precisión exacta en aritmética monetaria. En un sistema que maneja caja diaria con pagos parciales, los errores de redondeo de punto flotante son bugs con impacto financiero real. La integridad referencial entre órdenes, items, pagos y catálogo está garantizada a nivel de base de datos, no solo a nivel de aplicación.

### Cloudinary

Almacenamiento de fotos de tickets de lavado al seco. Tier gratuito de 25 GB suficiente para el volumen del negocio. El servidor de aplicaciones no gestiona archivos, lo que simplifica los deploys y elimina pérdida de datos al redesplegar.

*Alternativa descartada:* AWS S3 tiene mayor complejidad de configuración (IAM, bucket policies) sin beneficio adicional para una sola sucursal.

### Testing: Pytest + pytest-django

Las pruebas se concentran en `services.py` donde vive la lógica de negocio, testeable de forma aislada sin levantar el servidor HTTP.

Casos de prueba prioritarios:

- Cálculo de `total_amount` en órdenes con items mixtos (por prenda y por kilo).
- Validación de que un pago no excede el saldo pendiente de la orden.
- Transiciones de estado válidas e inválidas en `Order` y `OrderItem` (`django-fsm` rechaza las inválidas).
- Consistencia entre `dry_cleaning_status` de items y `status` de la orden padre.
- Cálculo de balance diario: total cobrado vs total pendiente.

---

## Infraestructura y Despliegue

### Hosting: Railway + Vercel

Backend Django y PostgreSQL en Railway. Frontend React en Vercel. Ambos con tier gratuito suficiente para el volumen de una sucursal.

| Servicio | Costo mensual |
|---|---|
| Railway (backend + PostgreSQL) | $0 – $5 USD |
| Vercel (frontend) | $0 |
| Cloudinary (imágenes) | $0 |
| **Total** | **$0 – $5 USD / mes (~$60/año)** |

El sistema actual basado en papel cuesta S/540/año (~$144 USD/año).

### Docker + Docker Compose

`Dockerfile` y `docker-compose.yml` garantizan paridad entre entorno local y producción. El entorno de desarrollo se levanta con un solo comando.

### CI/CD: GitHub Actions

Pipeline en cada push a `main`:

1. Instala dependencias y ejecuta `pytest`.
2. Si los tests pasan, construye la imagen Docker.
3. Despliega en Railway vía CLI.

Ningún código que rompa los tests llega a producción.

---

## Frontend

### React 19 + TypeScript

**Decisión técnica:** TypeScript previene errores en tiempo de desarrollo al tipar el modelo de dominio: una `Order` tiene `OrderItem[]` y `Payment[]`, con relaciones que el compilador valida antes de llegar a producción. El modelo de componentes de React maneja la actualización simultánea de métricas del dashboard, lista de órdenes e indicadores de pago sin recargas de página.

**Decisión de portafolio:** React es el framework frontend más demandado en el mercado laboral peruano y remoto. Para un proyecto que también funciona como carta de presentación profesional, la inversión en React + TypeScript tiene retorno directo en empleabilidad.

*Alternativa considerada:* Django templates + HTMX habría simplificado la arquitectura (sin JWT, sin CORS, sin deploy separado) y sería suficiente para el negocio. Se descartó conscientemente para maximizar el valor del proyecto como portafolio fullstack.

### Tailwind CSS v4

Diseño responsive adaptado a dos contextos de uso reales: registro de órdenes desde celular Android en el mostrador, revisión del dashboard desde desktop. El bundle final incluye solo las clases utilizadas.

### TailAdmin (MIT License)

Base estructural de componentes UI construidos con Tailwind, sin dependencias de librerías externas. El código vive en el repositorio, modificable sin restricciones de API de terceros.

Componentes adaptados al dominio del negocio: tabla de órdenes con columnas de estado y saldo pendiente, formulario de registro con items dinámicos y cálculo de total en tiempo real, dashboard con métricas operativas y financieras.

### ApexCharts

Visualización del dashboard: barras para ingresos por período, donut para distribución de métodos de pago (efectivo, Yape/Plin, transferencia). Integrado vía `react-apexcharts`.

### React Router v7 + Vite

React Router gestiona la navegación entre módulos sin recargas. Vite provee HMR en milisegundos durante el desarrollo.

---

## Repositorios

| Repositorio | Contenido |
|---|---|
| `laundry-ops-api` | Backend Django + DRF |
| `laundry-ops-web` | Frontend React + TypeScript |
| `laundry-ops-architecture` | Diagramas C4, ERD, user stories, decisiones técnicas |

---


*Última actualización: Febrero 2026*