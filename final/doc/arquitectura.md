# Especificación de Arquitectura y Alcance del Proyecto
**Materia:** Computación II  
**Proyecto:** Orquestador Distribuido de Agentes con Comunicación en Tiempo Real

---

## 1. Descripción Verbal de la Aplicación

### Problema que soluciona
En un sistema donde múltiples agentes de software cumplen roles específicos (recolectar datos de clientes, analizar conversaciones para asignar puntajes de interés y emitir alertas operativas), la comunicación directa punto a punto o sincrónica provoca bloqueos por espera de Entrada/Salida (I/O), pérdida de mensajes si un agente está ocupado y acoplamiento rígido que impide escalar el sistema.

### Funcionamiento de la solución Cliente-Servidor
Se plantea una arquitectura **Cliente-Servidor y de Procesamiento Distribuido** dividida en un **Plano de Control en Tiempo Real** y un **Plano de Trabajo Escalable**:

1. **Conexión y Comunicación Instantánea (Sockets TCP + Asincronismo de I/O):**
   * Los clientes son **Nodos Agentes** (iniciados por línea de comandos con `argparse` indicando su rol, por ejemplo `--role ingestor`, `--role scorer`, `--role operator`).
   * Se conectan de forma concurrente al **Servidor Orquestador Central (Hub)** mediante **Sockets `SOCK_STREAM` (TCP)**.
   * El Servidor Orquestador utiliza **`asyncio` (corrutinas y Event Loop)** para atender a decenas de agentes simultáneamente sin bloquearse. Cuando un agente envía un evento rápido, el servidor lo enruta al instante hacia el agente destino o en modo *broadcast*.

2. **Delegación de Tareas entre Agentes (Paralelismo y Cola Distribuida):**
   * Cuando un agente necesita designarle una tarea pesada a otro (ej.: el *Agente Ingestor* recibe una consulta de Meta CRM para **Nino Aventuras** y solicita que el *Agente Scorer* analice el historial de preguntas/respuestas, verifique promociones ofrecidas y calcule el puntaje del cliente), envía el pedido al Orquestador.
   * Para no congelar el socket ni el loop de `asyncio`, el Orquestador encola la tarea en **Redis (Broker)** para que sea ejecutada en paralelo por un pool de workers de **Celery**.
   * Al finalizar, el worker guarda los datos en la **Base de Datos**, deposita el resultado en **Redis (Backend)** y avisa al Orquestador, quien notifica en tiempo real por socket al agente solicitante.

3. **Auditoría y Telemetría Local (IPC: `os.fork`, `os.pipe`, `FIFO` y `signal`):**
   * Al iniciar, el Servidor Orquestador ejecuta **`os.fork()`** para crear un **Proceso Hijo Supervisor/Monitor** en la misma máquina.
   * Ambos se comunican mediante **`os.pipe()`** bajo el modelo **Productor-Consumidor**: el servidor escribe eventos de tráfico/latencia en el pipe y el hijo los procesa sin afectar el rendimiento de red.
   * A su vez, el proceso Monitor expone el estado de salud del sistema en un archivo **FIFO (named pipe)** en disco (`/tmp/orchestrator.fifo`) para permitir consultas locales en vivo.
   * El servidor captura señales **`SIGINT` y `SIGTERM` (`signal.signal()`)** para cerrar sockets limpiamente y ejecutar **`os.waitpid()`**, garantizando que no queden procesos zombies.

---

## 2. Gráfico de Arquitectura de la Aplicación

~~~text
+-------------------------------------------------------------------------------+
|                     NODOS AGENTES (Clientes TCP vía argparse)                 |
|                                                                               |
|  +----------------------+  +----------------------+  +---------------------+  |
|  | Agente Ingestor CRM  |  | Agente Analizador    |  | Consola Operador /  |  |
|  | (--role ingestor)    |  | (--role scorer)      |  | Agente Notificador  |  |
|  +----------+-----------+  +----------+-----------+  +----------+----------+  |
+-------------|-------------------------|-------------------------|-------------+
              |                         |                         |
              +-------------------------+-------------------------+
                                        |
                    Sockets STREAM (TCP) Concurrentes
             [Protocolo A2A-TCP: Header 4 Bytes + JSON-RPC 2.0]
                                        |
                                        v
+---------------------------+     +---------------------------------------------+
|  Proceso Monitor Local    |     |         SERVIDOR ORQUESTADOR (Hub)          |
|  (Proceso hijo: os.fork)  |<====|          (Python asyncio + sockets)         |
|                           | IPC |                                             |
| - Lee de os.pipe() (r, w) |     | - Registro dinámico de agentes y skills     |
| - Expone estado en FIFO   |     | - Ruteo instantáneo no bloqueante (await)   |
| - Control signal/waitpid  |     | - Captura SIGINT/SIGTERM (Cero Zombies)     |
+---------------------------+     +----------------------+----------------------+
                                                         |
                                      Encola tareas      | Recibe estado y
                                      delegadas (Broker) | resultados (Backend)
                                                         v
                                  +---------------------------------------------+
                                  |            REDIS (Broker & Backend)         |
                                  +----------------------+----------------------+
                                                         |
                                       Escalabilidad Horizontal de Workers
                                                         v
                                  +---------------------------------------------+
                                  |       POOL DE WORKERS (Celery + Beat)       |
                                  |                                             |
                                  | - Worker N: Ingesta y normalización de Lead |
                                  | - Worker N: Análisis Q&A, Promos y Scoring  |
                                  | - Worker N: Reporte diario de seguimiento   |
                                  +----------------------+----------------------+
                                                         |
                                                         v
                                  +---------------------------------------------+
                                  |       BASE DE DATOS (SQLite / PostgreSQL)   |
                                  |   Clientes, Historial Q&A, Promos y Tareas  |
                                  +---------------------------------------------+
~~~

---

## 3. Lista de Funcionalidades de cada Entidad

### A. Servidor Orquestador (`server/hub.py`)
* Parsea por CLI (`argparse`) el host, puerto, ruta de FIFO y configuración de Redis.
* Inicializa el socket TCP (`bind`, `listen`) y gestiona conexiones entrantes con `asyncio`.
* Mantiene en memoria el directorio de agentes conectados, sus roles y habilidades registradas (*Agent Cards*).
* Rutea mensajes instantáneos entre agentes conectados sin bloqueo de I/O.
* Despacha peticiones de tareas pesadas hacia Celery y devuelve el `task_id` inmediatamente.
* Crea al proceso Monitor vía `os.fork()`, le envía telemetría por `os.pipe()` y maneja señales (`SIGINT`/`SIGTERM`) con `os.waitpid()`.

### B. Nodo Agente Cliente (`agents/agent_node.py`)
* Se ejecuta desde terminal con `argparse` (`--host`, `--port`, `--role`, `--skill`, `--send-msg`, `--delegate`).
* Realiza el *handshake* de registro inicial ante el Orquestador declarando qué tareas sabe resolver.
* Escucha de manera asincrónica avisos instantáneos o resultados de tareas delegadas mientras permite emitir nuevos comandos.

### C. Proceso Monitor / Auditor IPC (`monitor/ipc_supervisor.py`)
* Proceso hijo creado por el servidor que lee continuamente del extremo de lectura `r` de `os.pipe()`.
* Contabiliza conexiones activas, bytes transferidos, tareas delegadas y errores de protocolo.
* Escribe resúmenes de estado bajo demanda a través de un archivo especial **FIFO (named pipe)** para diagnóstico local.

### D. Workers de Cola Distribuida (`workers/tasks.py`)
* Ejecutan en paralelo (multiproceso/distribuido con Celery) las tareas delegadas:
  1. **Ingesta de Lead:** Registra los datos del contacto entrante (ej. desde Meta CRM) en la base de datos.
  2. **Cálculo de Score y Promociones:** Evalúa las preguntas y respuestas del cliente, asocia promociones ofrecidas y calcula su nivel de prioridad comercial.
  3. **Agenda Diaria de Contacto:** Filtra los prospectos de mayor puntaje para notificar a quién se debe contactar en el día y qué detalle de la charla recordar.

---

## 4. Diseño Escalable (Adaptabilidad del Proyecto)
La arquitectura está diseñada en dos capas independientes (**Core de Orquestación** vs. **Skills de Agentes**), lo que permite escalar el alcance según requerimiento de la cátedra:
* **Escalabilidad de Agentes:** Se pueden conectar N nuevos tipos de agentes en caliente simplemente ejecutando nuevas instancias del cliente con distinto `--role`, sin reiniciar el servidor.
* **Escalabilidad de Cómputo:** Se pueden levantar múltiples contenedores de Celery (`docker compose up --scale worker=4`) para procesar tareas pesadas en paralelo real.
* **Extensibilidad de Seguridad y Red:** El transporte TCP admite envolver el socket con `ssl` (TLS/Certificados X.509) o incorporar un canal `SOCK_DGRAM` (UDP) para latidos (*heartbeats*) rápidos si se desea ampliar la complejidad.
