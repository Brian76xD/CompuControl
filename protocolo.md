# Protocolo de comunicación Master ↔ Esclavo

Documento de diseño del proyecto **CompuControl** (Control de múltiples PC dentro de un área específica).

Define cómo se comunican la PC del docente (**Master**) y las PCs de los alumnos (**esclavos**). Tanto el programa del Master como el del esclavo deben respetar este documento.

## 1. Transporte

- Protocolo de red: **TCP**.
- El **Master** es el servidor: escucha en `MASTER_PORT`.
- Los **esclavos** son los clientes: se conectan a `MASTER_IP:MASTER_PORT` (ambos definidos en el `.env`).
- Los esclavos inician siempre la conexión. El Master nunca busca equipos en la red.

## 2. Formato de los mensajes

- Cada mensaje es un **objeto JSON en una sola línea**, terminado con un salto de línea (`\n`).
- Codificación: **UTF-8**.
- Todo mensaje incluye el campo `"tipo"`, que indica qué mensaje es. Los demás campos dependen del tipo.
- Si llega un mensaje con un `tipo` desconocido o un JSON inválido, **se ignora** (no se cierra la conexión).

Ejemplo:

```
{"tipo": "LOCK", "mensaje": "Vista al frente"}
```

## 3. Mensajes

### Esclavo → Master

| Tipo | Campos | Descripción |
|---|---|---|
| `HELLO` | `equipo`, `codigo_aula` | Se presenta al conectarse. `equipo` es el nombre de la PC. |
| `ESTADO` | `bloqueado` (true/false) | Confirma si su pantalla está bloqueada. Se envía después de cada `LOCK` y `UNLOCK`. |
| `HEARTBEAT` | (ninguno) | Avisa que el equipo sigue conectado. |

### Master → Esclavo

| Tipo | Campos | Descripción |
|---|---|---|
| `WELCOME` | (ninguno) | Acepta al esclavo. |
| `RECHAZADO` | `motivo` | Rechaza al esclavo y cierra la conexión. |
| `LOCK` | `mensaje` | Ordena bloquear la pantalla mostrando el texto indicado. |
| `UNLOCK` | (ninguno) | Ordena desbloquear la pantalla. |

### Ejemplos

```
{"tipo": "HELLO", "equipo": "PC-AULA-01", "codigo_aula": "cambia-este-valor"}
{"tipo": "WELCOME"}
{"tipo": "RECHAZADO", "motivo": "codigo_invalido"}
{"tipo": "LOCK", "mensaje": "Vista al frente"}
{"tipo": "ESTADO", "bloqueado": true}
{"tipo": "HEARTBEAT"}
{"tipo": "UNLOCK"}
```

## 4. Flujo de conexión

1. El esclavo abre la conexión TCP con el Master.
2. El esclavo envía `HELLO` con su nombre de equipo y el `CODIGO_AULA`.
3. El Master compara el código:
   - Si **coincide**: responde `WELCOME` y agrega el equipo a su lista.
   - Si **no coincide**: responde `RECHAZADO` y cierra la conexión.
4. Si el docente tenía un bloqueo activo cuando el equipo se conectó, el Master le envía un `LOCK` de inmediato.

## 5. Flujo de bloqueo y desbloqueo

1. El docente presiona **Bloquear** en el panel (a un equipo o a todos).
2. El Master envía `LOCK` con el mensaje.
3. El esclavo muestra la ventana de bloqueo y responde `ESTADO` con `bloqueado: true`.
4. El docente presiona **Desbloquear**.
5. El Master envía `UNLOCK`.
6. El esclavo cierra la ventana de bloqueo y responde `ESTADO` con `bloqueado: false`.

## 6. Latido (heartbeat)

- Cada esclavo envía `HEARTBEAT` cada **5 segundos** (`INTERVALO_HEARTBEAT`).
- Si el Master no recibe nada de un equipo durante **15 segundos** (`TIMEOUT_HEARTBEAT`), lo marca como **desconectado** en el panel y cierra esa conexión.
- Cualquier mensaje recibido de un equipo cuenta también como señal de vida.

## 7. Reconexión

- Si el esclavo pierde la conexión, reintenta conectarse cada **3 segundos** (`INTERVALO_RECONEXION`) hasta lograrlo.
- Al reconectar repite el flujo de la sección 4 (envía `HELLO`).
- Si el Master ya tenía registrado un equipo con el mismo nombre, **reemplaza** la conexión anterior por la nueva.

## 8. Comportamiento sin conexión mientras está bloqueado

Decisión de diseño: **el equipo se mantiene bloqueado, pero con un tiempo límite de seguridad**.

- Si el esclavo pierde la conexión con el Master mientras está bloqueado, **sigue bloqueado** y sigue intentando reconectar.
- Si pasan **120 segundos** sin conexión (`TIMEOUT_SIN_MASTER`), el esclavo **se desbloquea solo**.
- Si se reconecta antes de ese tiempo, el Master decide: si el bloqueo sigue activo envía `LOCK`; si ya no, envía `UNLOCK`.

Motivo: evita que los alumnos queden atrapados si el Master se cierra o se cae la red, y a la vez dificulta que un alumno se libere simplemente desconectando el cable o el Wi-Fi.

## 9. Variables de configuración (.env)

| Variable | Ejemplo | Quién la usa |
|---|---|---|
| `MASTER_IP` | `192.168.1.10` | Esclavo |
| `MASTER_PORT` | `5050` | Master y esclavo |
| `CODIGO_AULA` | `cambia-este-valor` | Master y esclavo |
| `INTERVALO_HEARTBEAT` | `5` | Esclavo (segundos) |
| `TIMEOUT_HEARTBEAT` | `15` | Master (segundos) |
| `INTERVALO_RECONEXION` | `3` | Esclavo (segundos) |
| `TIMEOUT_SIN_MASTER` | `120` | Esclavo (segundos) |

## 10. Limitaciones conocidas (versión beta)

- Los mensajes **no van cifrados**; cualquiera en la red con herramientas de captura podría leerlos.
- El `CODIGO_AULA` identifica el aula, pero **no es un mecanismo de autenticación robusto**.
- No hay descubrimiento automático de equipos: cada esclavo debe conocer la IP del Master.
