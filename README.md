# Control de múltiples PC dentro de un área específica: CompuControl

Aplicación en red local que permite a un docente, desde su computadora (**Master**), bloquear y desbloquear las computadoras de los alumnos (**esclavos**) de un aula. Al bloquear, cada equipo muestra una pantalla completa con un mensaje (por ejemplo, "Vista al frente") que no se puede cerrar hasta que el docente lo decida, de forma similar a los bloqueos por tiempo de un café internet.

> **Estado:** versión beta / demostración. Alcance actual: 1 PC Master y al menos 2 PCs esclavo.

## Integrantes

- Campos Ayala Ana Gabriela		202421440
- López Rodríguez Brian Daly		202431778
- Luna Reyes Carmen			202516343
- Mendoza Cortés Valeria		202469974

## Requisitos

- Windows 10 u 11 (en el Master y en todos los esclavos)
- Python 3.10 o superior
- Todos los equipos conectados a la misma red local
- Git (para clonar el repositorio)

## Instalación

1. Clonar el repositorio:

   ```
   git clone https://github.com/USUARIO/NOMBRE-DEL-REPOSITORIO.git
   cd NOMBRE-DEL-REPOSITORIO
   ```

2. Crear y activar un entorno virtual:

   ```
   python -m venv venv
   venv\Scripts\activate
   ```

3. Instalar las dependencias:

   ```
   pip install -r requirements.txt
   ```

## Configuración (variables de entorno)

Los datos de configuración **no están escritos en el código**. Viven en un archivo `.env` que **no se sube al repositorio** (está incluido en `.gitignore`).

1. Copiar el archivo de ejemplo:

   ```
   copy .env.example .env
   ```

2. Abrir `.env` y llenar los valores:

   | Variable | Descripción |
   |---|---|
   | `MASTER_IP` | Dirección IP de la PC del docente (Master) |
   | `MASTER_PORT` | Puerto TCP en el que escucha el Master |
   | `CODIGO_AULA` | Código compartido entre el Master y sus esclavos, para evitar que se conecten equipos de otras aulas |

   El Master y los esclavos deben usar **el mismo** `CODIGO_AULA`.

**Importante:** nunca subir el archivo `.env` a GitHub. Solo se sube `.env.example`, que contiene valores de ejemplo.

## Uso

> Los nombres de los archivos pueden cambiar conforme avance el desarrollo.

**1. En la PC del docente (Master):**

```
python master\main.py
```

Se abre el panel con la lista de equipos conectados y los botones para bloquear y desbloquear.

**2. En cada PC de alumno (esclavo):**

```
python esclavo\main.py
```

El esclavo se conecta automáticamente al Master. Cuando el docente presiona **Bloquear**, la pantalla del alumno se cubre con el mensaje indicado hasta que el docente presione **Desbloquear**.

## Cómo funciona

- Arquitectura **cliente-servidor** sobre TCP (sockets de Python).
- El **Master** actúa como servidor: acepta conexiones y envía comandos (`LOCK`, `UNLOCK`).
- Los **esclavos** se conectan al Master (no al revés), por lo que no es necesario escanear la red ni listar equipos de otras aulas.
- La ventana de bloqueo se construye con Tkinter: pantalla completa, siempre al frente y sin botón de cerrar, con los atajos de teclado comunes deshabilitados (Alt+F4, Esc, etc.).
- El esclavo arranca automáticamente al iniciar sesión en Windows mediante el Programador de tareas.

## Estructura de carpetas

```
control-aula/
├── master/            Programa del docente
├── esclavo/           Programa del alumno
├── comun/             Código compartido (protocolo de mensajes)
├── docs/              Diagramas y documentación
├── .env.example       Plantilla de variables de entorno
├── .gitignore
├── requirements.txt
└── README.md
```

## Limitaciones conocidas

- No es posible bloquear por software la combinación **Ctrl+Alt+Supr**, por lo que un usuario podría abrir el Administrador de tareas.
- Un usuario con permisos suficientes podría cerrar el proceso del esclavo desde el Administrador de tareas.
- El esclavo arranca con el Programador de tareas al iniciar sesión, no como servicio de Windows (un servicio real no puede mostrar ventanas en el escritorio del usuario). Queda como mejora futura.
- No hay descubrimiento automático de equipos: cada esclavo debe tener configurada la IP del Master.