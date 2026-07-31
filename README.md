# 🤖 AgendaBot - Asistente Personal Automatizado

AgendaBot es un sistema avanzado de asistencia personal que opera íntegramente a través de Telegram. Diseñado con una arquitectura modular orientada a microservicios en **n8n** y utilizando **Google Sheets** como base de datos relacional, el bot permite gestionar el día a día del usuario de forma conversacional y estructurada.

Este proyecto aplica fundamentos sólidos de lógica de programación, manejo de estado en entornos sin estado (stateless) y metodologías ágiles para un desarrollo escalable.

---

## 🏗️ Arquitectura del Sistema

El bot abandona el enfoque tradicional de "un solo flujo gigante" y adopta un modelo de **Enrutador Principal y Submódulos Especializados**[cite: 8]. Esto garantiza un código limpio, fácil de mantener y tolerante a fallos.

### 1. Ingesta de Datos y Enrutador (`AgendaBot_Main`)
El núcleo del sistema. Como n8n Community sin URL pública no puede recibir webhooks directos, el sistema implementa un mecanismo de **Polling**[cite: 8]:
* **Sondeo Constante:** Consulta la API de Telegram cada 20 segundos[cite: 8].
* **Control de Offset:** Utiliza una variable en la memoria estática del flujo (`$getWorkflowStaticData`) para garantizar que ningún mensaje se procese dos veces[cite: 8].
* **Validación de Identidad:** Verifica si el usuario existe en la hoja `USUARIOS`[cite: 8]. Si la variable `AUTO_REGISTRO` está activa, el primer usuario en interactuar recibe automáticamente el rol de `ADMIN`, asegurando un proceso de despliegue sin intervención manual[cite: 8].
* **Router Inteligente:** Analiza la `pantalla_actual` y el `texto` enviado para derivar la petición al submódulo correcto a través del nodo *Execute Workflow*, sin intentar "adivinar" intenciones de manera imprecisa[cite: 8].

### 2. Manejo de Estado (`AgendaBot_Sessions`)
Telegram y HTTP son protocolos sin estado. Para mantener una conversación guiada (wizards), el sistema utiliza la hoja `SESSIONS`[cite: 11]:
* Cada usuario tiene exactamente una fila (creada o actualizada mediante `appendOrUpdate`)[cite: 11].
* Se almacena la `pantalla_actual`, el `paso_actual` y un objeto JSON llamado `datos_parciales` que actúa como caché temporal de las respuestas del usuario[cite: 11].
* **Caducidad:** Si pasan más de 6 horas sin interacción, la sesión caduca automáticamente por seguridad y el usuario regresa al menú principal[cite: 8].

### 3. Auditoría Estricta (`AgendaBot_Logs`)
La "caja negra" del sistema. Cada interacción genera una fila inmutable (solo *append*) en la hoja `LOGS`[cite: 7]. 
* Registra el `timestamp`, el `telegram_user`, la `pantalla`, la `opcion_elegida` y el `resultado` exacto del proceso[cite: 7].
* Fundamental para depurar errores (`FECHA_INVALIDA`, `DOBLE_RESERVA`, etc.) y analizar el uso de las opciones[cite: 7].

---

## 🧩 Módulos y Funcionalidades en Detalle

Cada submódulo es un archivo JSON independiente que recibe un contrato estándar de entrada y devuelve un contrato estricto de salida con la acción a realizar (`NINGUNA`, `CREAR`, `ACTUALIZAR`)[cite: 3].

### 📅 Módulo Agenda (`AgendaBot_Agenda`)
Gestión integral del calendario del usuario.
* **Wizard de 6 pasos:** Solicita fecha, hora, nombre, motivo y canal (Presencial, Virtual, Llamada) antes de guardar[cite: 3].
* **Validaciones Robustas:** Impide agendar en fechas pasadas, valida formatos (`AAAA-MM-DD` y `HH:MM`) y bloquea **dobles reservas** si ya existe una cita activa en la misma franja[cite: 3].
* **Mutaciones:** Permite reprogramar (cambiando fecha/hora), cancelar o marcar como completadas las citas existentes[cite: 3].

### ✅ Módulo Tareas (`AgendaBot_Tareas`)
Administración de pendientes diarios.
* Las tareas se categorizan por **Prioridad** (Alta, Media, Baja) y pueden tener o no una fecha objetivo[cite: 9].
* Los estados manejados son: `PENDIENTE`, `EN PROGRESO`, `COMPLETADA` y `CANCELADA`.

### 🔁 Módulo Hábitos (`AgendaBot_Habitos`)
Construcción y seguimiento de rutinas recurrentes[cite: 5].
* Soporta frecuencias: Diario, Días hábiles, Semanal, Mensual[cite: 5].
* **Eliminación Lógica (Soft Delete):** Al borrar un hábito, no se destruye la fila en la base de datos, sino que su estado pasa a `ELIMINADO`[cite: 5]. Esto preserva la integridad del historial.

### 📝 Módulo Listas (`AgendaBot_Listas`)
Sistema relacional (Maestro-Detalle) dentro de Google Sheets[cite: 6].
* Gestiona dos hojas conectadas: `LISTAS` (cabecera) e `ITEMS_LISTA` (detalle de los elementos) mediante una llave única `id_lista`[cite: 6].
* Permite crear listas temáticas (Compras, Tareas, Ideas) y gestionar individualmente el estado de cada ítem (agregar, marcar completado, eliminar)[cite: 6].

### 🔔 Módulo Recordatorios (`AgendaBot_Recordatorios`)
Módulo de *solo lectura* que unifica `CITAS` y `TAREAS` mediante un nodo Merge[cite: 9].
* Ofrece proyecciones inmediatas: Próximas 24 horas, Próximos 7 días, Agenda de Hoy[cite: 9].
* Analiza y filtra tareas para alertar sobre aquellas vencidas, para hoy, o para la semana en curso[cite: 9].

### 📊 Módulo Reportes (`AgendaBot_Reportes`)
Motor analítico del asistente[cite: 10].
* Lee las 5 hojas de negocio en cadena, filtrando exclusivamente por el `creado_por` del usuario solicitante[cite: 10].
* Dibuja **barras de progreso visuales (`█`)** directamente en texto plano calculando porcentajes matemáticos de los estados, eliminando la necesidad de APIs gráficas externas[cite: 10].

### 🛡️ Módulo Administrador (`AgendaBot_Admin`)
Panel de control protegido por una doble barrera de seguridad (validado en el Router y nuevamente en el submódulo)[cite: 2, 8].
* Permite auditar el sistema leyendo los 20 logs más recientes[cite: 2].
* Facilita la gestión de usuarios: consultar registrados, activar bloqueados, desactivar y gestionar accesos[cite: 2].
* **Protección anti-bloqueo:** Impide que el administrador se desactive a sí mismo o que se degrade al rol `USER` si no hay otro administrador en el sistema[cite: 2].

---

## ⚙️ Tareas en Segundo Plano (Automatizaciones)

El archivo `AgendaBot_Automatizaciones` contiene trabajos programados (CRON) que le dan proactividad al bot[cite: 4].

1. **Resumen Diario (07:00 AM):**
   - Se ejecuta todos los días a primera hora[cite: 4].
   - Compila un reporte personalizado para cada usuario habilitado, resumiendo sus citas del día, tareas abiertas o vencidas, y hábitos activos[cite: 4].
2. **Aviso de Citas Próximas (Cada 15 min):**
   - Rastrea continuamente la hoja de `CITAS`[cite: 4].
   - Si detecta un evento a menos de 60 minutos, envía una notificación push al usuario[cite: 4].
   - Utiliza `$getWorkflowStaticData` para marcar la alerta como `avisada` y evitar que el usuario reciba mensajes duplicados en la siguiente consulta de 15 minutos[cite: 4].

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
2. **Importación:** Cargar todos los archivos `.json` de la carpeta `workflows/` en un entorno n8n.
3. **Reconexión de Nodos:** 
   - Abrir el flujo `AgendaBot_Main` y reconectar manualmente cada nodo *Execute Workflow* (Agenda, Tareas, etc.) a su respectivo submódulo importado[cite: 8].
4. **Credenciales:**
   - Configurar la autenticación de Google Sheets y reemplazar la cadena temporal `GOOGLE_SHEET_ID` por el ID real del documento en *todos* los submódulos.
   - Insertar el token del Bot Father de Telegram en la URL del nodo HTTP `Telegram getUpdates`[cite: 8].
5. **Puesta en Marcha:** 
   - Activar (`Published`) todos los submódulos de apoyo.
   - Activar los flujos `AgendaBot_Main` y `AgendaBot_Automatizaciones`.
   - Enviar `/start` al bot desde Telegram; la primera persona será coronada automáticamente como `ADMIN`[cite: 8].

---
*Desarrollado y estructurado con un enfoque modular, aplicando conceptos de desarrollo de interfaces conversacionales y automatización backend.*

**Jaime Andrés Naranjo Pinzón**  
*Frontend Designer & Web Developer*  
*Campusland · Colombia*