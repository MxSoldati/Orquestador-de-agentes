# 🚀 Orquestador Distribuido de Agentes en Tiempo Real (`A2A-TCP`)
**Trabajo Final Integrador - Computación II**  
**Autor:** Máximo Soldati  
**Estado actual:** Fase 1 - Definición de Arquitectura y Protocolo (En revisión docente)

---

## 📌 Introducción
Este proyecto consiste en un **Servidor Orquestador Distribuido (Hub)** de propósito general y altamente escalable, diseñado bajo arquitectura Cliente-Servidor para coordinar múltiples agentes de software autónomos.

El sistema resuelve dos desafíos críticos en entornos multi-agente:
1. **Comunicación instantánea de baja latencia:** Los nodos agentes mantienen conexiones persistentes vía **Sockets `SOCK_STREAM` (TCP)** gestionadas con **asincronismo de I/O (`asyncio`)**, permitiendo el enrutamiento de eventos y alertas en tiempo real sin bloquear el servidor.
2. **Delegación y escalabilidad horizontal de tareas pesadas:** Cuando un agente delega una tarea de cómputo intensivo o de consulta externa a otro rol, el Orquestador desacopla su ejecución enviándola a un **Pool Distribuido de Workers (`Celery` + `Redis`)**, permitiendo escalar la cantidad de nodos procesadores a demanda sin afectar el ciclo pregunta-respuesta del socket.

### 🎯 Caso de Uso Aplicado (Plugin de Negocio: *Nino Aventuras*)
Sobre el motor agéntico escalable se monta un pipeline real de gestión comercial:
* **Agente Ingestor:** Captura prospectos entrantes desde canales externos (Meta CRM).
* **Agente Analizador / Scorer:** Analiza el historial de preguntas/respuestas, registra promociones ofrecidas y calcula un puntaje de prioridad (*Lead Scoring*).
* **Agente Notificador / Operador:** Recibe alertas instantáneas de clientes prioritarios y genera la agenda diaria de seguimiento.

---

## 🛠️ Stack de Herramientas de Computación II

| Requisito de la Cátedra | Implementación en el Proyecto |
| :--- | :--- |
| **Sockets Concurrentes** | Sockets `AF_INET` / `SOCK_STREAM` (TCP) con soporte multi-cliente simultáneo. |
| **Protocolo de Aplicación** | **`A2A-TCP`**: Framing binario de 4 bytes (prefijo de longitud) + Payload **JSON-RPC 2.0** inspirado en el estándar *Agent-to-Agent (A2A)*. |
| **Asincronismo de I/O** | Event loop cooperativo con **`asyncio`** (`async/await`, `create_task`) evitando bloqueos por GIL y *race conditions*. |
| **Mecanismos de IPC y Señales** | Proceso hijo auditor creado con **`os.fork()`**, comunicado por **`os.pipe()`** (Productor-Consumidor), exposición en disco vía **`FIFO` (named pipe)** y apagado limpio con **`signal.signal()`** + **`os.waitpid()`** (sin procesos zombies). |
| **Cola de Tareas Distribuidas** | **`Celery`** con **`Redis`** como *Broker* de mensajes y *Backend* de resultados. |
| **Argumentos por CLI** | Configuración dinámica de roles, puertos, IPs y acciones mediante **`argparse`**. |
| **Escalabilidad y Despliegue** | Contenedores **`Docker`** (`docker-compose.yml`) y persistencia en base de datos relacional (**SQLite / PostgreSQL**). |

---

## 📂 Documentación del Proyecto (`final/doc/`)
* [Especificación de Arquitectura y Entidades](./final/doc/arquitectura.md)
* [Especificación del Protocolo A2A-TCP](./final/doc/protocolo_a2a.md)
* [Hoja de Ruta y Mejoras Futuras (TODO)](./final/TODO.md)
