# Hoja de Ruta y Mejoras Futuras (TODO)

## 1. Roadmap de Desarrollo Incremental (Por Issues)
- [ ] **Issue #1 (Documentación y Protocolo):** Definir arquitectura general en `doc/` y especificar el protocolo `A2A-TCP` (Framing de 4 bytes + JSON-RPC 2.0). *Requisito previo a codificar.*
- [ ] **Issue #2 (Capa de Protocolo y CLI):** Implementar empaquetador/desempaquetador TCP sobre streams y configuración por consola mediante `argparse`.
- [ ] **Issue #3 (Orquestador Asincrónico):** Desarrollar el Servidor Hub con `asyncio` y clientes agentes concurrentes con registro de capacidades (`Agent Cards`) y mensajería instantánea.
- [ ] **Issue #4 (Mecanismos de IPC y Señales):** Incorporar proceso hijo auditor mediante `os.fork()`, comunicación productor-consumidor vía `os.pipe()`, exposición de métricas en archivo `FIFO` y captura de `SIGINT`/`SIGTERM` con `os.waitpid()`.
- [ ] **Issue #5 (Cola Distribuida y Persistencia):** Integrar `Celery` con `Redis` (Broker/Backend) y base de datos SQLite/PostgreSQL para las tareas de *Nino Aventuras* (Ingesta Meta CRM, Lead Scoring de Q&A y Agenda Diaria).
- [ ] **Issue #6 (Despliegue y Documentación Final):** Orquestar contenedores con `docker-compose.yml` y redactar `INSTALL.md` e `INFO.md`.

---

## 2. Mejoras y Nuevas Características para Futuras Versiones (Post-Examen)
- [ ] **Seguridad en Transporte (SSL/TLS):** Incorporar certificados X.509 (`openssl`) en el socket `SOCK_STREAM` con el módulo `ssl` de Python.
- [ ] **Alta Disponibilidad del Hub:** Permitir múltiples instancias del Servidor Orquestador sincronizadas mediante Redis Pub/Sub.
- [ ] **Dashboard Web en Tiempo Real:** Incorporar una interfaz visual que consuma el archivo FIFO o se conecte como un agente observador.
