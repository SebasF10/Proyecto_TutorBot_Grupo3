# TutorBot — Sistema de Asesorías Académicas

Automatización construida en **n8n** que conecta estudiantes con tutores a través de **Telegram**, usando **Google Sheets** como base de datos. Reemplaza la coordinación manual por correo/mensajes con un flujo guiado que asigna tutor y horario automáticamente, y cierra las tutorías vencidas sin intervención manual.

## Integrantes

- Keynner Alonso Sanchez Ortiz
- Sebastian Feo Murillo
- Nicolas David Naranjo Barrios

---

## Índice

- [Qué problema resuelve](#qué-problema-resuelve)
- [Arquitectura general](#arquitectura-general)
- [Modelo de datos](#modelo-de-datos-google-sheets)
- [Cómo funciona el flujo](#cómo-funciona-el-flujo)
- [Envío de mensajes multicanal](#envío-de-mensajes-multicanal)
- [Finalización automática de tutorías](#finalización-automática-de-tutorías)
- [Estados de una tutoría](#estados-de-una-tutoría)
- [Detalle de los nodos — workflow principal](#detalle-de-los-nodos--workflow-principal)
- [Detalle de los nodos — Enviar Mensaje](#detalle-de-los-nodos--enviar-mensaje)
- [Detalle de los nodos — Finalizar Tutorías](#detalle-de-los-nodos--finalizar-tutorías)
- [Comandos disponibles en Telegram](#comandos-disponibles-en-telegram)
- [Instalación y configuración](#instalación-y-configuración)
- [Pruebas locales con ngrok](#pruebas-locales-con-ngrok)
- [Novedades recientes](#novedades-recientes)
- [Estructura del repositorio](#estructura-del-repositorio)

---

# Qué problema resuelve

Coordinar asesorías académicas suele hacerse por correo o mensajes sueltos: no hay agenda centralizada, se cruzan horarios y no queda trazabilidad de quién atendió a quién. **TutorBot** automatiza todo el ciclo:

1. El estudiante pide una tutoría desde Telegram.
2. El sistema busca los tutores con la materia y el horario disponibles.
3. El estudiante elige uno de la lista, se bloquea el horario y se notifica a ambas partes.
4. Cuando la fecha y hora de la tutoría ya pasaron, el sistema la cierra automáticamente.

**Objetivos del proyecto:**
- Motor de búsqueda que asocia automáticamente materia → tutor → horario libre.
- Interfaz conversacional en Telegram para autogestión (solicitar, elegir, cancelar).
- Control automático de estados de la tutoría, incluido el cierre por vencimiento.
- Validación de disponibilidad en tiempo real (sin dobles reservas).

---

# Arquitectura general

El proyecto son **tres workflows de n8n** que trabajan juntos sobre la misma base de datos en Google Sheets.

| Workflow | Disparador | Rol |
|---|---|---|
| **TutorBot (principal)** | Telegram Trigger | Contiene todo el wizard conversacional: sesión, menús, búsqueda de tutor, confirmación y creación de la tutoría. |
| **TutorBot - Enviar Mensaje** | Execute Workflow Trigger (llamado por el principal) | Sub-workflow reutilizable que centraliza el envío de cada respuesta, eligiendo el canal correcto. |
| **TutorBot - Finalizar Tutorías** | Schedule Trigger (cada hora) | Revisa la hoja `TUTORIAS` y marca como `Finalizada` cualquier tutoría cuya fecha y hora ya pasaron. |

| Módulo | Rol |
|---|---|
| **Interfaz Telegram** | Único punto de contacto del estudiante: registrarse, elegir materia por menú numérico, elegir tutor entre varias opciones y recibir notificaciones. |
| **Motor n8n** | Orquesta la lógica de los tres workflows: sesión, validaciones, asignación de tutores y cierre automático. |
| **Google Sheets (TutorBot_DB)** | Base de datos compartida por los tres workflows: tutores, disponibilidad, tutorías y sesiones activas. |

El nodo **`Normalizar Mensaje`** del workflow principal detecta si el mensaje viene de Telegram, Discord o WhatsApp Cloud API y lo homogeniza a un mismo formato interno (`canal`, `user_id`, `texto`, `nombre`). Hoy el disparador activo es Telegram, pero el motor ya está preparado para sumar otros canales sin rehacer la lógica de negocio.

---

# Modelo de datos (Google Sheets)

La base `TutorBot_DB` (mismo `documentId` en los tres workflows) tiene 4 hojas:

| Hoja | Columnas clave | Para qué sirve |
|---|---|---|
| **TUTORES** | `id_tutor`, `nombre`, `especialidad_materias`, `estado` | Catálogo de tutores activos/inactivos y qué materias dictan. |
| **DISPONIBILIDAD** | `id_dispo`, `id_tutor`, `dia_semana`, `hora_inicio`, `hora_fin`, `estado` | Horarios libres/ocupados de cada tutor. |
| **TUTORIAS** | `id_tutoria`, `id_estudiante`, `id_tutor`, `materia`, `fecha`, `hora`, `estado`, `fecha_finalizacion` | Registro histórico de cada asesoría solicitada. |
| **SESIONES** | `telegram_user`, `canal`, `pantalla_actual`, `paso_actual`, `datos_parciales` | "Memoria" del bot: en qué paso del wizard va cada usuario, por qué canal escribió y qué datos ya capturó. |

![Modelo de datos en Google Sheets](assets/modelo-datos.png)

---

# Cómo funciona el flujo

El flujo más importante es **"Solicitar Tutoría"**, un wizard de varios pasos donde el bot va guiando al estudiante y guardando su avance en la hoja `SESIONES`.

![Diagrama de flujo](assets/diagrama-de-flujo.png)

Vista completa del canvas en n8n:

![Canvas completo del workflow principal](assets/canvas-completo.png)

**Resumen del recorrido:**

1. **Trigger + normalización** — llega el mensaje de Telegram y se homogeniza.
2. **¿Hay sesión activa?** — si el usuario ya estaba a mitad de un flujo, se enruta directo a la pantalla donde iba (vía `Switch - Enrutar Según Pantalla`); si no, se revisa si escribió el comando para solicitar tutoría.
3. **Menú de materias** — se leen los tutores activos, se extraen las materias únicas y se muestran como opciones numeradas.
4. **Validación de materia** — si la opción no existe, se le vuelve a pedir; si es válida, se pasa a pedir la fecha.
5. **Validación de fecha** — se espera formato `YYYY-MM-DD`.
6. **Búsqueda de tutores** — se filtra `TUTORES` por la materia elegida (comparando contra `especialidad_materias`) y estado `Activo`, y se cruza con `DISPONIBILIDAD` para esa fecha.
7. **Sin disponibilidad** — si no hay tutores libres, se avisa al estudiante y se limpia la sesión.
8. **Consolidar candidatos** — el nodo `Consolidar Mensaje - Tutores Disponibles` arma una lista numerada con todos los tutores disponibles y sus horarios.
9. **Selección del estudiante** — el estudiante responde con el **número del tutor que prefiere** (o escribe `cancelar`). El nodo `Interpretar Respuesta` valida la elección contra la lista de candidatos.
10. **Creación de la tutoría** — con una selección válida, se agrega el registro en `TUTORIAS` (estado `Asignada`) usando el tutor y horario elegidos, y se marca la disponibilidad como `Ocupado`.
11. **Notificaciones** — se avisa al estudiante y, si el tutor tiene un chat de Telegram vinculado, también se le notifica a él.
12. **Cierre** — se limpia la sesión y el usuario queda libre para iniciar otro flujo.

Si la respuesta es `cancelar` o un número inválido, el sistema cancela el proceso o vuelve a preguntar sin perder el contexto ya capturado. Como el registro en `TUTORIAS` solo se crea al final (paso 10), cancelar durante el wizard no deja ninguna fila huérfana en la hoja.

> **Cambio respecto a versiones anteriores:** antes el bot sugería un único tutor y el estudiante solo confirmaba con "sí/no". Ahora el bot muestra **todos** los tutores disponibles en una lista numerada y el estudiante elige directamente cuál prefiere. Además, la búsqueda de tutores ahora sí filtra por la materia solicitada.

---

# Envío de mensajes multicanal

Todo el envío de mensajes está desacoplado en un **sub-workflow independiente**: `TutorBot - Enviar Mensaje`. El workflow principal nunca llama directamente a la API de Telegram — en cambio, cada nodo `Enviar Mensaje - *` invoca este sub-workflow por `Execute Workflow`, pasándole `canal`, `destinatario` y `mensaje`.

![Canvas del sub-workflow Enviar Mensaje](assets/canvas-enviar-mensaje.png)

**Cómo funciona:**

1. **`Cuando se Llama el Sub-flujo`** recibe el payload (`canal`, `destinatario`, `mensaje`) del workflow principal.
2. **`Switch - Por Canal`** revisa el campo `canal` y lo enruta a `telegram`, `discord` o `whatsapp` (con una salida de respaldo si no coincide con ninguno).
3. **`Enviar por Telegram`** toma la rama de Telegram y envía el mensaje al `chatId` correspondiente usando la credencial del bot.

Las ramas de `discord` y `whatsapp` ya existen en el switch pero todavía no tienen nodos conectados — quedan listas para cuando el proyecto se extienda a esos canales, sin tener que tocar el workflow principal.

---

# Finalización automática de tutorías

Este workflow, **`TutorBot - Finalizar Tutorías`**, corre solo, sin que nadie escriba nada en Telegram, y se encarga de cerrar las tutorías que ya pasaron.

![Canvas del workflow Finalizar Tutorías](assets/canvas-finalizar-tutorias.png)

**Cómo funciona:**

1. **`Schedule Trigger`** inicia automáticamente el workflow en horarios programados (cada hora). A diferencia del bot de Telegram, aquí no espera que un usuario envíe mensajes — el sistema se ejecuta solo.
2. **`Leer Todas las Tutorias`** consulta la hoja donde están registradas todas las tutorías del sistema y lee cada fila para analizar su estado, porque el sistema necesita revisar todas las reservas existentes antes de decidir cuáles ya vencieron.
3. **`Detectar Tutorias Vencidas`** (Code) analiza todas las tutorías obtenidas desde Google Sheets y compara sus fechas con la fecha actual, para identificar cuáles ya pasaron y todavía siguen marcadas como activas o programadas.
4. **`Update row in sheet`** actualiza las filas correspondientes a las tutorías vencidas encontradas en el nodo anterior: las tutorías que ya ocurrieron quedan marcadas como `Finalizada`, junto con `fecha_finalizacion`.

Este workflow es el que realmente mueve una tutoría de `Asignada` a `Finalizada` — no depende de que el estudiante o el tutor confirmen nada por chat.

---

# Estados de una tutoría

Según lo que efectivamente escriben los workflows en la hoja `TUTORIAS`, el ciclo de vida real es:

- **Asignada** — se crea en este estado cuando el estudiante elige un tutor de la lista (paso 10 del wizard).
- **Finalizada** — la pone automáticamente `TutorBot - Finalizar Tutorías` cuando la fecha y hora programadas ya pasaron. Se guarda también `fecha_finalizacion`.
- **Cancelada** — no es un estado que se escriba en `TUTORIAS`; ocurre solo durante el wizard, antes de que exista el registro. Si el estudiante cancela o no hay disponibilidad, simplemente no se crea ninguna fila.

No existe un estado intermedio `Confirmada` en la implementación actual: el paso de `Asignada` a `Finalizada` es automático por horario, no por una confirmación manual adicional.

---

# Detalle de los nodos — workflow principal

El archivo `Tutorbot.json` contiene **42 nodos**.

## 1. Entrada y sesión

**`Telegram Trigger`**
Detecta cuando llega un mensaje por Telegram e inicializa el flujo de n8n. Almacena los datos iniciales y los pasa al resto del workflow.
![Telegram Trigger](assets/nodes/telegram-trigger.png)

**`Normalizar Mensaje`** (Code)
Toma la información que da el nodo anterior y crea un formato mucho más sencillo para el resto del flujo.
![Normalizar Mensaje](assets/nodes/normalizar-mensaje.png)

**`Buscar Sesión Activa`** (Google Sheets)
Lee los datos normalizados de la persona que está hablando con el bot para verificar si ya había iniciado un proceso anteriormente o es la primera vez que escribe.
![Buscar Sesion Activa](assets/nodes/buscar-sesion-activa.png)

**`IF - Existe Sesión`**
Compara el resultado del nodo anterior y revisa si el usuario tenía datos de una conversación previa o si es una nueva conversación.
![IF - Existe Sesion](assets/nodes/if-existe-sesion.png)

**`IF - Es Comando Solicitar`**
Evita que cualquier mensaje haga que el sistema empiece una reserva — solo unos comandos puntuales inician el wizard.
![IF - Es Comando Solicitar](assets/nodes/if-es-comando-solicitar.png)

**`Enviar Mensaje - Ignorar`** (Execute Workflow)
Si el usuario no escribe el comando correcto, manda un mensaje de error.
![Enviar Mensaje - Ignorar](assets/nodes/enviar-mensaje-ignorar.png)

## 2. Enrutamiento

**`Switch - Enrutar Según Pantalla`**
Valida en qué paso se había quedado el usuario anteriormente y lo envía para que continúe justo donde se quedó (`inicio`, `esperando_materia`, `esperando_fecha`, `esperando_confirmacion`).
![Switch - Enrutar Segun Pantalla](assets/nodes/switch-enrutar-segun-pantalla.png)

## 3. Selección de materia

**`Leer Tutores Activos`** (Google Sheets)
Lee todos los tutores que están registrados como activos y los almacena en un JSON para que otro nodo los lea.
![Leer Tutores Activos](assets/nodes/leer-tutores-activos.png)

**`Extraer Materias Únicas`** (Code)
Extrae las materias que se encuentran en la hoja y se las pasa al siguiente nodo.
![Extraer Materias Unicas](assets/nodes/extraer-materias-unicas.png)

**`Enviar Mensaje - Menú Materias`** (Execute Workflow)
Recibe los datos del nodo anterior y los envía en formato de mensaje bien diseñado para el estudiante.
![Enviar Mensaje - Menu Materias](assets/nodes/enviar-mensaje-menu-materias.png)

**`Guardar Sesión - Esperando Materia`** (Google Sheets)
Guarda la materia elegida y modifica las celdas correspondientes en la base de datos.
![Guardar Sesion - Esperando Materia](assets/nodes/guardar-sesion-esperando-materia.png)

**`Validar Opción Materia`** (Code)
Nodo tipo código que valida la opción de materia elegida y revisa si esa opción existe en la base de datos.
![Validar Opcion Materia](assets/nodes/validar-opcion-materia.png)

**`IF - Opción Válida`**
Verifica la comprobación del nodo anterior — si la opción existe o no — y decide cómo prosigue el flujo.
![IF - Opcion Valida](assets/nodes/if-opcion-valida.png)

**`Enviar Mensaje - Opción Inválida`** (Execute Workflow)
Le manda un mensaje de error al usuario informando que la opción elegida no existe.
![Enviar Mensaje - Opcion Invalida](assets/nodes/enviar-mensaje-opcion-invalida.png)

**`Enviar Mensaje - Pedir Fecha`** (Execute Workflow)
Envía un mensaje al estudiante pidiéndole que seleccione o escriba la fecha en la que desea recibir la tutoría.
![Enviar Mensaje - Pedir Fecha](assets/nodes/enviar-mensaje-pedir-fecha.png)

**`Actualizar Sesión - Esperando Fecha`** (Google Sheets)
Después de enviar el mensaje, actualiza la hoja `SESIONES` para indicar que el sistema ahora está esperando una fecha.
![Actualizar Sesion - Esperando Fecha](assets/nodes/actualizar-sesion-esperando-fecha.png)

## 4. Selección de fecha y búsqueda de tutores

**`Validar Fecha`** (Code)
Recibe la fecha que escribió el estudiante y verifica que tenga el formato correcto y que pueda utilizarse para buscar disponibilidad, guardando el resultado.
![Validar Fecha](assets/nodes/validar-fecha.png)

**`IF - Fecha Válida`**
Toma el resultado del nodo anterior y valida ese resultado para decidir si la fecha es utilizable.
![IF - Fecha Valida](assets/nodes/if-fecha-valida.png)

**`Enviar Mensaje - Fecha Inválida`** (Execute Workflow)
Envía un mensaje indicando que la fecha ingresada no cumple el formato esperado.
![Enviar Mensaje - Fecha Invalida](assets/nodes/enviar-mensaje-fecha-invalida.png)

**`Leer Tutores Por Materia`** (Google Sheets)
Consulta la hoja donde están registrados los tutores y obtiene únicamente aquellos que enseñan la materia seleccionada por el estudiante.
![Leer Tutores Por Materia](assets/nodes/leer-tutores-por-materia.png)

**`Leer Disponibilidad`** (Google Sheets)
Lee la hoja donde están registrados los horarios disponibles de los tutores.
![Leer Disponibilidad](assets/nodes/leer-disponibilidad.png)

**`Buscar Tutor Disponible`** (Code)
Combina la información obtenida por los dos nodos anteriores y, con esa información, busca un tutor de esa materia con esa fecha disponible, devolviendo los que cumplen con eso.
![Buscar Tutor Disponible](assets/nodes/buscar-tutor-disponible.png)

**`IF - Hay Tutores Disponibles`**
Recibe la lista generada por `Buscar Tutor Disponible` y verifica si contiene al menos un tutor.
![IF - Hay Tutores Disponibles](assets/nodes/if-hay-tutores-disponibles.png)

**`Enviar Mensaje - Sin Disponibilidad`** (Execute Workflow)
Informa al estudiante que no fue posible encontrar un tutor disponible para la materia y la fecha seleccionadas.
![Enviar Mensaje - Sin Disponibilidad](assets/nodes/enviar-mensaje-sin-disponibilidad.png)

**`Limpiar Sesión - Sin Disponibilidad`** (Google Sheets)
Deja la conversación preparada para que, si el estudiante desea volver a intentarlo, pueda iniciar una nueva solicitud desde el principio.
![Limpiar Sesion - Sin Disponibilidad](assets/nodes/limpiar-sesion-sin-disponibilidad.png)

**`Consolidar Mensaje - Tutores Disponibles`** (Code)
Recibe la lista de tutores disponibles encontrada en `Buscar Tutor Disponible` y construye un mensaje organizado para enviárselo al estudiante.
![Consolidar Mensaje - Tutores Disponibles](assets/nodes/consolidar-mensaje-tutores-disponibles.png)

**`Enviar Mensaje - Tutores Disponibles`** (Execute Workflow)
Envía al estudiante el mensaje generado por el nodo anterior. En ese mensaje aparecen los tutores disponibles junto con sus horarios para que el estudiante pueda escoger uno.
![Enviar Mensaje - Tutores Disponibles](assets/nodes/enviar-mensaje-tutores-disponibles.png)

**`Actualizar Sesión - Esperando Confirmación`** (Google Sheets)
Después de mostrar los tutores disponibles, actualiza la hoja `SESIONES` indicando que ahora está esperando la respuesta del estudiante.
![Actualizar Sesion - Esperando Confirmacion](assets/nodes/actualizar-sesion-esperando-confirmacion.png)

## 5. Confirmación y creación de la tutoría

**`Interpretar Respuesta`** (Code)
Recibe el mensaje que acaba de escribir el estudiante después de que el bot le mostró la lista de tutores, y lo interpreta para que el switch de confirmación pueda funcionar.
![Interpretar Respuesta](assets/nodes/interpretar-respuesta.png)

**`Switch - Confirmación`**
Enruta según la respuesta interpretada: selección válida de un tutor, cancelación, o respuesta inválida.
![Switch - Confirmacion](assets/nodes/switch-confirmacion.png)

**`Crear Tutoría`** (Google Sheets)
Crea oficialmente la tutoría en la hoja correspondiente de Google Sheets. Con este nodo la tutoría queda registrada en la base de datos.
![Crear Tutoria](assets/nodes/crear-tutoria.png)

**`Actualizar Disponibilidad Ocupado`** (Google Sheets)
Una vez creada la tutoría, cambia el estado del horario del tutor para que no se pueda volver a reservar.
![Actualizar Disponibilidad Ocupado](assets/nodes/actualizar-disponibilidad-ocupado.png)

**`Enviar Mensaje - Éxito Estudiante`** (Execute Workflow)
Informa al estudiante que la tutoría fue creada correctamente, por Telegram.
![Enviar Mensaje - Exito Estudiante](assets/nodes/enviar-mensaje-exito-estudiante.png)

## 6. Notificación al tutor y cierre de sesión

**`Extraer Datos Tutor Notificación`** (Code)
Obtiene la información necesaria para enviar una notificación al tutor.
![Extraer Datos Tutor Notificacion](assets/nodes/extraer-datos-tutor-notificacion.png)

**`IF - Tutor Tiene Chat ID`**
Comprueba si el tutor tiene registrado un Chat ID de Telegram: si lo tiene, el sistema puede enviarle el mensaje automáticamente; si no, no es posible enviar la notificación.
![IF - Tutor Tiene Chat ID](assets/nodes/if-tutor-tiene-chat-id.png)

**`Enviar Mensaje - Notificar Tutor`** (Execute Workflow)
Envía al tutor el aviso de que se le asignó una nueva tutoría.
![Enviar Mensaje - Notificar Tutor](assets/nodes/enviar-mensaje-notificar-tutor.png)

**`Limpiar Sesión - Éxito`** (Google Sheets)
El sistema limpia esta sesión para que el usuario pueda iniciar otra desde cero.
![Limpiar Sesion - Exito](assets/nodes/limpiar-sesion-exito.png)

**`Enviar Mensaje - Cancelación`** (Execute Workflow)
Informa al estudiante que la solicitud de tutoría ha sido cancelada correctamente, cerrando la conversación de forma clara.
![Enviar Mensaje - Cancelacion](assets/nodes/enviar-mensaje-cancelacion.png)

**`Limpiar Sesión - Cancelación`** (Google Sheets)
Actualiza la hoja `SESIONES` para finalizar la conversación del estudiante.
![Limpiar Sesion - Cancelacion](assets/nodes/limpiar-sesion-cancelacion.png)

**`Enviar Mensaje - Respuesta Inválida`** (Execute Workflow)
Informa al estudiante que la respuesta enviada no pudo ser interpretada por el sistema, porque no corresponde a ninguna de las alternativas disponibles.
![Enviar Mensaje - Respuesta Invalida](assets/nodes/enviar-mensaje-respuesta-invalida.png)

**`Enviar Mensaje - Error General`** (Execute Workflow)
Avisa al estudiante ante cualquier error inesperado del flujo.
![Enviar Mensaje - Error General](assets/nodes/enviar-mensaje-error-general.png)

**`Limpiar Sesión - Error`** (Google Sheets)
Limpia la sesión del usuario tras un error, para que pueda empezar de nuevo.
![Limpiar Sesion - Error](assets/nodes/limpiar-sesion-error.png)

---

# Detalle de los nodos — Enviar Mensaje

El archivo `TutorBot - Enviar Mensaje.json` contiene **3 nodos**.

**`Cuando se Llama el Sub-flujo`** (Execute Workflow Trigger)
Recibe `canal`, `destinatario` y `mensaje` desde el workflow principal cada vez que hay que enviar un mensaje.
![Cuando se Llama el Sub-flujo](assets/nodes/enviar-mensaje/cuando-se-llama-el-sub-flujo.png)

**`Switch - Por Canal`**
Despacha el envío según el campo `canal` recibido: `telegram`, `discord` o `whatsapp`.
![Switch - Por Canal](assets/nodes/enviar-mensaje/switch-por-canal.png)

**`Enviar por Telegram`**
Envía el mensaje usando el bot de Telegram, al `chatId` del destinatario. Es la única rama con un nodo conectado por ahora.
![Enviar por Telegram](assets/nodes/enviar-mensaje/enviar-por-telegram.png)

---

# Detalle de los nodos — Finalizar Tutorías

El archivo `TutorBot - Finalizar Tutorias.json` contiene **4 nodos**. Este workflow está **activo**, corriendo cada hora de forma independiente.

**`Schedule Trigger`**
Inicia automáticamente el workflow en horarios programados. A diferencia del bot de Telegram, aquí no espera que un usuario envíe mensajes: el sistema se ejecuta solo.
![Schedule Trigger](assets/nodes/finalizar-tutorias/schedule-trigger.png)

**`Leer Todas las Tutorias`** (Google Sheets)
Consulta la hoja donde están registradas todas las tutorías del sistema y lee cada fila para analizar su estado, porque el sistema necesita revisar todas las reservas existentes antes de decidir cuáles ya vencieron.
![Leer Todas las Tutorias](assets/nodes/finalizar-tutorias/leer-todas-las-tutorias.png)

**`Detectar Tutorias Vencidas`** (Code)
Analiza todas las tutorías obtenidas desde Google Sheets y compara sus fechas con la fecha actual, para identificar cuáles ya pasaron y todavía siguen marcadas como activas o programadas.
![Detectar Tutorias Vencidas](assets/nodes/finalizar-tutorias/detectar-tutorias-vencidas.png)

**`Update row in sheet`** (Google Sheets)
Actualiza las filas correspondientes a las tutorías vencidas encontradas en el nodo anterior. Las tutorías que ya ocurrieron quedan marcadas como finalizadas.
![Update row in sheet](assets/nodes/finalizar-tutorias/update-row-in-sheet.png)

---

# Comandos disponibles en Telegram

| Acción | Cómo se dispara |
|---|---|
| Solicitar tutoría | Comando de inicio del wizard (detectado en `IF - Es Comando Solicitar`) |
| Elegir materia | Responder con el número mostrado en el menú |
| Elegir fecha | Escribir la fecha en formato `YYYY-MM-DD` |
| Elegir tutor | Responder con el número del tutor de la lista de disponibles |
| Cancelar | Escribir `no`, `n` o `cancelar` en cualquier paso de confirmación |

---

# Instalación y configuración

1. **Importar los tres workflows** en n8n, en este orden:
   - `TutorBot - Enviar Mensaje.json`
   - `TutorBot - Finalizar Tutorias.json`
   - `Tutorbot.json` (el principal, que llama al primero por `Execute Workflow`)
2. **Enlazar el sub-workflow de mensajería**: en los nodos `Execute Workflow` del workflow principal (`Enviar Mensaje - *`), verifica que el `workflowId` apunte al `TutorBot - Enviar Mensaje` que acabas de importar en tu instancia (el ID cambia entre instancias de n8n).
3. **Credenciales necesarias**:
   - Telegram (Bot Token) en `Telegram Trigger` (workflow principal) y en `Enviar por Telegram` (sub-workflow de mensajería).
   - Google Sheets (OAuth2) en todos los nodos `n8n-nodes-base.googleSheets` de los tres workflows.
4. **Base de datos**: crea una copia de `TutorBot_DB` en Google Sheets con las 4 hojas descritas arriba y reemplaza el `documentId` en los nodos de Google Sheets de los tres workflows por el de tu copia.
5. **Activar** los tres workflows. `TutorBot - Finalizar Tutorías` debe quedar activo para que el cierre automático funcione; si está desactivado, las tutorías se quedarán en `Asignada` para siempre.

---

# Pruebas locales con ngrok

Si corres n8n en tu máquina (`localhost`), Telegram no puede enviarle mensajes directamente porque necesita una URL pública con HTTPS. Para probar el bot en local se usó **[ngrok](https://ngrok.com/)**, que crea un túnel público hacia tu servidor local y te da un dominio temporal para pruebas.

**Pasos básicos:**

1. Instalar ngrok (o usar el binario portable): https://ngrok.com/download
2. Levantar n8n localmente (por defecto en el puerto `5678`):
   ```
   n8n start
   ```
3. Abrir el túnel apuntando al puerto de n8n:
   ```
   ngrok http 5678
   ```
4. ngrok entrega una URL pública tipo `https://xxxxx.ngrok-free.app`. Copiarla.
5. Configurar esa URL como webhook del bot de Telegram (n8n lo hace automáticamente al activar el nodo `Telegram Trigger` si le indicas la URL pública en `N8N_WEBHOOK_URL` / `WEBHOOK_URL`, o puedes registrarlo manualmente con la API de Telegram: `https://api.telegram.org/bot<TOKEN>/setWebhook?url=<URL_DE_NGROK>/webhook/...`).
6. Probar el bot enviándole mensajes desde Telegram; las peticiones deberían llegar a tu n8n local a través del túnel.

> Las URLs gratuitas de ngrok cambian cada vez que reinicias el túnel, así que hay que volver a registrar el webhook en Telegram cada vez que se reinicia ngrok (a menos que tengas un dominio fijo contratado).

---

# Novedades recientes

- **El proyecto ya está finalizado y funcional**: los tres workflows están completos y `TutorBot - Finalizar Tutorías` pasó de estar desactivado a **activo**, cerrando tutorías automáticamente cada hora.
- **Capturas reales de los 49 nodos** incorporadas al README, organizadas por workflow, más las vistas de canvas completo de cada uno.
- **Diagramas propios en draw.io** (`diagrams-source/`) del workflow principal y de `Finalizar Tutorías`, con las descripciones de cada nodo en palabras del equipo.
- **Nuevo workflow `TutorBot - Finalizar Tutorías`**: cierra automáticamente las tutorías vencidas cada hora, sin depender de que alguien confirme nada por chat.
- **Filtro de tutores corregido**: `Leer Tutores Por Materia` ahora sí filtra por la materia que el estudiante eligió.
- **Unificación del campo de mensaje**: `Consolidar Mensaje - Tutores Disponibles` ahora envía el texto en el campo `mensaje` en vez de `texto`, para que coincida con lo que espera el sub-workflow `TutorBot - Enviar Mensaje`.

---

# Estructura del repositorio

```
Proyecto_TutorBot/
├── README.md                              # Este archivo
├── Tutorbot.json                          # Workflow principal (wizard conversacional)
├── TutorBot - Enviar Mensaje.json         # Sub-workflow de envio de mensajes por canal
├── TutorBot - Finalizar Tutorias.json     # Workflow programado que cierra tutorias vencidas
├── diagrams-source/                       # Diagramas propios editables (draw.io)
│   ├── Tutor_Bot_v2.drawio                 # Diagrama del workflow principal
│   └── Tutor_bot_finalizar.drawio          # Diagrama del workflow Finalizar Tutorias
└── assets/
    ├── estructura.png                     # Placeholder: arquitectura general (falta)
    ├── modelo-datos.png                   # Placeholder: hojas de Google Sheets (falta)
    ├── estados-tutoria.png                # Placeholder: diagrama de estados (falta)
    ├── ngrok-tunnel.png                   # Placeholder: consola de ngrok (falta)
    ├── diagrama-de-flujo.png              # Diagrama del flujo principal (draw.io)
    ├── canvas-completo.png                # Captura del canvas del workflow principal
    ├── canvas-enviar-mensaje.png          # Captura del canvas de Enviar Mensaje
    ├── canvas-finalizar-tutorias.png      # Captura del canvas de Finalizar Tutorias
    └── nodes/                             # Captura individual de cada nodo (42 del principal)
        ├── telegram-trigger.png
        ├── normalizar-mensaje.png
        ├── buscar-sesion-activa.png
        ├── if-existe-sesion.png
        ├── if-es-comando-solicitar.png
        ├── enviar-mensaje-ignorar.png
        ├── switch-enrutar-segun-pantalla.png
        ├── leer-tutores-activos.png
        ├── extraer-materias-unicas.png
        ├── enviar-mensaje-menu-materias.png
        ├── guardar-sesion-esperando-materia.png
        ├── validar-opcion-materia.png
        ├── if-opcion-valida.png
        ├── enviar-mensaje-opcion-invalida.png
        ├── enviar-mensaje-pedir-fecha.png
        ├── actualizar-sesion-esperando-fecha.png
        ├── validar-fecha.png
        ├── if-fecha-valida.png
        ├── enviar-mensaje-fecha-invalida.png
        ├── leer-tutores-por-materia.png
        ├── leer-disponibilidad.png
        ├── buscar-tutor-disponible.png
        ├── if-hay-tutores-disponibles.png
        ├── enviar-mensaje-sin-disponibilidad.png
        ├── limpiar-sesion-sin-disponibilidad.png
        ├── consolidar-mensaje-tutores-disponibles.png
        ├── enviar-mensaje-tutores-disponibles.png
        ├── actualizar-sesion-esperando-confirmacion.png
        ├── interpretar-respuesta.png
        ├── switch-confirmacion.png
        ├── crear-tutoria.png
        ├── actualizar-disponibilidad-ocupado.png
        ├── enviar-mensaje-exito-estudiante.png
        ├── extraer-datos-tutor-notificacion.png
        ├── if-tutor-tiene-chat-id.png
        ├── enviar-mensaje-notificar-tutor.png
        ├── limpiar-sesion-exito.png
        ├── enviar-mensaje-cancelacion.png
        ├── limpiar-sesion-cancelacion.png
        ├── enviar-mensaje-respuesta-invalida.png
        ├── enviar-mensaje-error-general.png
        ├── limpiar-sesion-error.png
        ├── enviar-mensaje/                # 3 nodos del sub-workflow Enviar Mensaje
        │   ├── cuando-se-llama-el-sub-flujo.png
        │   ├── switch-por-canal.png
        │   └── enviar-por-telegram.png
        └── finalizar-tutorias/             # 4 nodos del workflow Finalizar Tutorias
            ├── schedule-trigger.png
            ├── leer-todas-las-tutorias.png
            ├── detectar-tutorias-vencidas.png
            └── update-row-in-sheet.png
```

Quedan 4 imágenes pendientes por generar (`estructura.png`, `modelo-datos.png`, `estados-tutoria.png`, `ngrok-tunnel.png`) — todo lo demás (los 49 nodos, los 3 canvas y el diagrama de flujo) ya está incluido con capturas reales.

---

# Resultado esperado

- Reducción drástica del tiempo de asignación de tutorías frente al proceso manual.
- Trazabilidad total: historial de quién solicitó, quién atendió y cuándo terminó cada asesoría.
- Cierre automático de tutorías vencidas, sin depender de que nadie confirme nada manualmente.
- Escalable a cientos de tutores y estudiantes simultáneos.
- Experiencia guiada paso a paso, con opción de elegir entre varios tutores disponibles, sin necesidad de manuales para el estudiante.
