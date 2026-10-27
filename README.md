# Orquestador istribuido de Agentes en Tiempo Real

**Trabajo Final Integrador - Computación II**  
**Caso de Aplicación Real:** Sistema de Gestión, Scoring y Seguimiento de Leads (*Nino Aventuras*)

---

## 📌 Introducción y Propósito
Este proyecto implementa una arquitectura **Cliente-Servidor distribuida y asincrónica** diseñada para coordinar múltiples agentes de software especializados. 

Su objetivo es resolver dos problemas críticos en el procesamiento de información en vivo (aplicado a la recepción de consultas comerciales desde Meta CRM para *Nino Aventuras*):
1. **Comunicación instantánea sin bloqueos:** Permitir que distintos nodos agentes (Ingestores, Analizadores, Notificadores y Operadores) intercambien eventos y alertas al instante a través de un **Servidor Orquestador Central (Hub)**.
2. **Delegación de tareas pesadas:** Desacoplar las tareas costosas (análisis de historial de conversación, cálculo de *Lead Scoring*, cruce de promociones y generación de agendas diarias de contacto) enviándolas a una **cola de tareas distribuidas**, evitando congelar el ciclo de atención del servidor.

---

## 🛠️ Herramientas y Conceptos de Computación II Aplicados

El diseño integra de forma práctica los ejes centrales de la materia:

* **Sockets Concurrentes (`SOCK_STREAM` / TCP):** Conexión confiable de múltiples nodos agentes simultáneos hacia el Servidor Orquestador.
* **Asincronismo de I/O (`asyncio`):** Manejo cooperativo de corrutinas (`async`/`await`) en el servidor central para enrutar mensajes entre decenas de agentes sin bloqueos por espera de red ni condiciones de carrera (*race conditions*).
* **Mecanismos de IPC Local (`os.fork`, `os.pipe`, `FIFO`, `signal`):** Creación de un proceso hijo supervisor encargado de recolectar telemetría y auditar el tráfico del servidor mediante tuberías en memoria y archivos FIFO en disco, con manejo limpio de señales (`SIGINT`/`SIGTERM` y `waitpid`) para evitar procesos zombies.
* **Cola de Tareas Distribuidas (`Celery` + `Redis`):** Uso de Redis como *broker* de mensajes y *backend* de resultados para que un pool de workers procese en paralelo el análisis de clientes y el armado de reportes diarios.
* **Argumentos por Línea de Comandos (`argparse`):** Configuración dinámica de hosts, puertos, roles de cada agente y comandos operativos desde la terminal.

---

## 📂 Estructura del Proyecto

* `doc/`: Documentación conceptual previa al desarrollo (Descripción verbal, diagrama de arquitectura y detalle de entidades).
* `README.md`: Introducción general y guía de uso de la aplicación.
* `INSTALL.md`: Instrucciones para preparar el entorno y desplegar los servicios.
* `INFO.md`: Justificación teórica de las decisiones de arquitectura, concurrencia e IPC.
* `TODO.md`: Hoja de ruta del desarrollo incremental y mejoras futuras.




<img src="https://user-images.githubusercontent.com/74038190/212746035-d5c61762-973c-44c0-aec7-887f3b7690e3.git"width="500">
<br><br>
