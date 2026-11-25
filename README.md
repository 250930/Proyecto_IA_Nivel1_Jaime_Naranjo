# 🤖 AgendaBot - Asistente Personal Automatizado

AgendaBot es un sistema avanzado de asistencia personal que opera íntegramente a través de Telegram. Diseñado con una arquitectura modular orientada a microservicios en **n8n** y utilizando **Google Sheets** como base de datos relacional, el bot permite gestionar el día a día del usuario de forma conversacional y estructurada.

**Tecnologías:** n8n · Telegram Bot API · Google Sheets · JavaScript (nodos Code)

Este proyecto aplica fundamentos sólidos de lógica de programación, manejo de estado en entornos sin estado (stateless) y metodologías ágiles para un desarrollo escalable.

---

## 🏗️ Arquitectura del Sistema

El bot abandona el enfoque tradicional de "un solo flujo gigante" y adopta un modelo de **Enrutador Principal y Submódulos Especializados**. Esto garantiza un código limpio, fácil de mantener y tolerante a fallos.

### 1. Ingesta de Datos y Enrutador (`AgendaBot_Main`)
El núcleo del sistema. Como n8n Community sin URL pública no puede recibir webhooks directos, el sistema implementa un mecanismo de **Polling**:
* **Sondeo Constante:** Consulta la API de Telegram cada 20 segundos.
* **Control de Offset:** Utiliza una variable en la memoria estática del flujo (`$getWorkflowStaticData`) para garantizar que ningún mensaje se procese dos veces.
* **Validación de Identidad:** Verifica si el usuario existe en la hoja `USUARIOS`. Si la variable `AUTO_REGISTRO` está activa, el primer usuario en interactuar recibe automáticamente el rol de `ADMIN`, asegurando un proceso de despliegue sin intervención manual.
* **Router Inteligente:** Analiza la `pantalla_actual` y el `texto` enviado para derivar la petición al submódulo correcto a través del nodo *Execute Workflow*, sin intentar "adivinar" intenciones de manera imprecisa.

### 2. Manejo de Estado (`AgendaBot_Sessions`)
Telegram y HTTP son protocolos sin estado. Para mantener una conversación guiada (wizards), el sistema utiliza la hoja `SESSIONS`:
* Cada usuario tiene exactamente una fila (creada o actualizada mediante `appendOrUpdate`).
* Se almacena la `pantalla_actual`, el `paso_actual` y un objeto JSON llamado `datos_parciales` que actúa como caché temporal de las respuestas del usuario.
* **Caducidad:** Si pasan más de 6 horas sin interacción, la sesión caduca automáticamente por seguridad y el usuario regresa al menú principal.

### 3. Auditoría Estricta (`AgendaBot_Logs`)
La "caja negra" del sistema. Cada interacción genera una fila inmutable (solo *append*) en la hoja `LOGS`. 
* Registra el `timestamp`, el `telegram_user`, la `pantalla`, la `opcion_elegida` y el `resultado` exacto del proceso.
* Fundamental para depurar errores (`FECHA_INVALIDA`, `DOBLE_RESERVA`, etc.) y analizar el uso de las opciones.

---

## 🧩 Módulos y Funcionalidades en Detalle

Cada submódulo es un archivo JSON independiente que recibe un contrato estándar de entrada y devuelve un contrato estricto de salida con la acción a realizar (`NINGUNA`, `CREAR`, `ACTUALIZAR`).

### 📅 Módulo Agenda (`AgendaBot_Agenda`)
Gestión integral del calendario del usuario.
* **Wizard de 6 pasos:** Solicita fecha, hora, nombre, motivo y canal (Presencial, Virtual, Llamada) antes de guardar.
* **Validaciones Robustas:** Impide agendar en fechas pasadas, valida formatos (`AAAA-MM-DD` y `HH:MM`) y bloquea **dobles reservas** si ya existe una cita activa en la misma franja.
* **Mutaciones:** Permite reprogramar (cambiando fecha/hora), cancelar o marcar como completadas las citas existentes.

### ✅ Módulo Tareas (`AgendaBot_Tareas`)
Administración de pendientes diarios.
* Las tareas se categorizan por **Prioridad** (Alta, Media, Baja) y pueden tener o no una fecha objetivo.
* Los estados manejados son: `PENDIENTE`, `EN PROGRESO`, `COMPLETADA` y `CANCELADA`.

### 🔁 Módulo Hábitos (`AgendaBot_Habitos`)
Construcción y seguimiento de rutinas recurrentes.
* Soporta frecuencias: Diario, Días hábiles, Semanal, Mensual.
* **Eliminación Lógica (Soft Delete):** Al borrar un hábito, no se destruye la fila en la base de datos, sino que su estado pasa a `ELIMINADO`. Esto preserva la integridad del historial.

### 📝 Módulo Listas (`AgendaBot_Listas`)
Sistema relacional (Maestro-Detalle) dentro de Google Sheets.
* Gestiona dos hojas conectadas: `LISTAS` (cabecera) e `ITEMS_LISTA` (detalle de los elementos) mediante una llave única `id_lista`.
* Permite crear listas temáticas (Compras, Tareas, Ideas) y gestionar individualmente el estado de cada ítem (agregar, marcar completado, eliminar).

### 🔔 Módulo Recordatorios (`AgendaBot_Recordatorios`)
Módulo de *solo lectura* que unifica `CITAS` y `TAREAS` mediante un nodo Merge.
* Ofrece proyecciones inmediatas: Próximas 24 horas, Próximos 7 días, Agenda de Hoy.
* Analiza y filtra tareas para alertar sobre aquellas vencidas, para hoy, o para la semana en curso.

### 📊 Módulo Reportes (`AgendaBot_Reportes`)
Motor analítico del asistente.
* Lee las 5 hojas de negocio en cadena, filtrando exclusivamente por el `creado_por` del usuario solicitante.
* Dibuja **barras de progreso visuales (`█`)** directamente en texto plano calculando porcentajes matemáticos de los estados, eliminando la necesidad de APIs gráficas externas.

### 📈 Reporte de Productividad por Usuario
Se integró una nueva opción avanzada dentro del menú de **Reportes (Opción 6)**. Este módulo ahora consolida la información de las hojas `CITAS`, `TAREAS` y `LOGS` para generar una métrica de rendimiento grupal e individual.

**Funcionalidades del reporte:**
* **Lectura Multi-Hoja:** Procesa de manera encadenada todo el historial de la base de datos sin depender de webhooks externos.
* **Métricas Individuales:** Agrupa los datos alfabéticamente por usuario (identificador de Telegram).
  * *Citas:* Total registradas, completadas y canceladas.
  * *Tareas:* Total registradas, hechas y pendientes.
  * *Interacciones:* Conteo total de acciones del usuario registradas en los logs.
* **Control de Nulos:** Los usuarios sin historial en tareas o citas aparecen con valores absolutos en `0`, garantizando que el reporte no se rompa por falta de registros.
* **Resumen General:** Destaca al usuario más activo del sistema y las sumatorias globales.

### 🛡️ Módulo Administrador (`AgendaBot_Admin`)
Panel de control protegido por una doble barrera de seguridad (validado en el Router y nuevamente en el submódulo).
* Permite auditar el sistema leyendo los 20 logs más recientes.
* Facilita la gestión de usuarios: consultar registrados, activar bloqueados, desactivar y gestionar accesos.
* **Protección anti-bloqueo:** Impide que el administrador se desactive a sí mismo o que se degrade al rol `USER` si no hay otro administrador en el sistema.

---

## ⚙️ Tareas en Segundo Plano (Automatizaciones)

El flujo `AgendaBot_Automatizaciones` contiene trabajos programados (CRON) que le dan proactividad al bot. *(Este flujo no está incluido en el repositorio.)*

1. **Resumen Diario (07:00 AM):**
   - Se ejecuta todos los días a primera hora.
   - Compila un reporte personalizado para cada usuario habilitado, resumiendo sus citas del día, tareas abiertas o vencidas, y hábitos activos.
2. **Aviso de Citas Próximas (Cada 15 min):**
   - Rastrea continuamente la hoja de `CITAS`.
   - Si detecta un evento a menos de 60 minutos, envía una notificación push al usuario.
   - Utiliza `$getWorkflowStaticData` para marcar la alerta como `avisada` y evitar que el usuario reciba mensajes duplicados en la siguiente consulta de 15 minutos.

---

## 🗄️ Estructura de la Base de Datos (Google Sheets)

El documento principal consta de 8 hojas con generación automática de identificadores correlativos (ej. `CITA-001`, `HAB-005`):
* `USUARIOS`: `telegram_user`, `nombre`, `rol`, `permitido`
* `SESSIONS`: `telegram_user`, `pantalla_actual`, `paso_actual`, `datos_parciales`, `timestamp`
* `CITAS`: `id_cita`, `fecha`, `hora`, `nombre`, `motivo`, `canal`, `estado`, `creado_por`
* `TAREAS`: `id_tarea`, `titulo`, `prioridad`, `fecha_objetivo`, `estado`, `creado_por`
* `HABITOS`: `id_habito`, `nombre`, `frecuencia`, `hora_recordatorio`, `estado`
* `LISTAS`: `id_lista`, `nombre_lista`, `tipo`, `creado_por`
* `ITEMS_LISTA`: `id_item`, `id_lista`, `item`, `estado`
* `LOGS`: `timestamp`, `telegram_user`, `pantalla`, `opcion_elegida`, `resultado`

---

## 🚀 Guía Rápida de Despliegue

1. **Base de Datos:** Duplicar la plantilla de Google Sheets con las 8 hojas vacías.
2. **Importación:** Importar en n8n todos los archivos `AgendaBot_*.json` de este repositorio.
3. **Reconexión de Nodos:** 
   - Abrir el flujo `AgendaBot_Main` y reconectar manualmente cada nodo *Execute Workflow* (Agenda, Tareas, etc.) a su respectivo submódulo importado.
4. **Credenciales:**
   - Configurar la autenticación de Google Sheets y reemplazar la cadena temporal `GOOGLE_SHEET_ID` por el ID real del documento en *todos* los submódulos.
   - Insertar el token del Bot Father de Telegram en la URL del nodo HTTP `Telegram getUpdates`.
5. **Puesta en Marcha:** 
   - Activar (`Published`) todos los submódulos de apoyo.
   - Activar el flujo `AgendaBot_Main`.
   - Enviar `/start` al bot desde Telegram; la primera persona será coronada automáticamente como `ADMIN`.

## 📸 Evidencias

- [Base de datos de ejemplo en Google Sheets](https://docs.google.com/spreadsheets/d/168N7ux4ov96joBmO9tR3EjgjedoVW3tTYxZHGC5nRLc/edit?usp=sharing)
- [Documento de evidencias](https://docs.google.com/document/d/1fcQzwiUv6E2rMIs3VLmT_whzD1mYIZdG5C5eWdmywHY/edit?usp=sharing)
- [Capturas del reporte de productividad en Telegram](https://docs.google.com/document/d/1p_2Yu7gUaCfnLZ2O890NpM22WrxgOnctFaL71DN0cAQ/edit?usp=sharing)

---
*Desarrollado y estructurado con un enfoque modular, aplicando conceptos de desarrollo de interfaces conversacionales y automatización backend.*

**Jaime Andrés Naranjo Pinzón**  
*Frontend Designer & Web Developer*  
*Campuslands · Colombia*  
GitHub: [@250930](https://github.com/250930)
