# Especificación del Protocolo de Capa de Aplicación (`A2A-TCP`)

Para garantizar una comunicación determinista y libre de errores sobre sockets `SOCK_STREAM` (TCP), el sistema define un protocolo propio en dos niveles: **Entramado Binario (Framing)** y **Mensajería JSON-RPC 2.0 inspirada en A2A (Agent-to-Agent)**.

---

## 1. Nivel de Transporte: Entramado sobre Flujo TCP (Framing)
Dado que TCP es un protocolo orientado a flujo de bytes continuo (donde una llamada a `recv()` puede recibir fragmentos incompletos o múltiples mensajes concatenados), todo paquete comienza con un **encabezado fijo de 4 bytes**:

~~~text
+--------------------------------------+----------------------------------------+
| HEADER: 4 Bytes (uint32 Big-Endian)  | PAYLOAD: N Bytes (UTF-8 JSON-RPC 2.0)  |
| Indica longitud exacta N del JSON    | Cuerpo del mensaje codificado          |
+--------------------------------------+----------------------------------------+
~~~

* **Lectura segura en `asyncio`:** El receptor lee exactamente 4 bytes (`readexactly(4)`), desempaqueta el entero sin signo N (`struct.unpack('!I', header)`) y luego lee exactamente N bytes (`readexactly(N)`).

---

## 2. Nivel de Aplicación: Métodos del Protocolo (JSON-RPC 2.0 / A2A)

### A. Registro de Agente (`agent/register`)
Enviado por el cliente apenas establece la conexión TCP para anunciar su rol y habilidades (*Agent Card*):
~~~json
{
  "jsonrpc": "2.0",
  "method": "agent/register",
  "params": {
    "agent_id": "scorer-node-01",
    "role": "scorer",
    "skills": ["calculate_lead_score", "daily_followup_report"]
  },
  "id": 1
}
~~~

### B. Comunicación Instantánea entre Agentes (`message/instant`)
Enrutado en memoria por el Servidor Orquestador (`asyncio`) hacia el rol o agente destino en tiempo real:
~~~json
{
  "jsonrpc": "2.0",
  "method": "message/instant",
  "params": {
    "from_role": "ingestor",
    "target_role": "operator",
    "priority": "HIGH",
    "message": "Nuevo contacto recibido desde Meta CRM solicitando presupuesto."
  }
}
~~~

### C. Delegación de Tarea entre Agentes (`tasks/send`)
Solicita que otro rol ejecute una tarea pesada a través de la cola distribuida (Celery + Redis):
~~~json
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "params": {
    "target_role": "scorer",
    "skill": "calculate_lead_score",
    "payload": {
      "lead_name": "Laura Pérez",
      "questions": ["Precio para 15 chicos", "Tienen descuento en semana?"],
      "promo_offered": "PROMO_SEMANA_15"
    }
  },
  "id": 2
}
~~~

### D. Respuesta Asincrónica y Notificación de Resultado (`tasks/status`)
1. **Acuse inmediato del Orquestador:** Devuelve `"state": "submitted"` y el `task_id` de Celery para no bloquear el socket.
2. **Aviso de finalización (*Push*):** Cuando el worker de Celery termina, el Orquestador envía el resultado al agente interesado:
~~~json
{
  "jsonrpc": "2.0",
  "method": "tasks/status",
  "params": {
    "task_id": "a1b2-c3d4",
    "state": "completed",
    "result": {
      "score": 90,
      "action": "Contactar hoy y recordar PROMO_SEMANA_15"
    }
  }
}
~~~
