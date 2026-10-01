# Requirements Document

## Introduction

UNDC-Empleo es un asistente organizacional de empleabilidad y seguimiento a egresados para la Escuela Profesional de Ingeniería de Sistemas de la Universidad Nacional de Cañete (UNDC). El sistema traduce el perfil académico de estudiantes y egresados a competencias comparables con el mercado laboral, expone las brechas frente a vacantes concretas mediante cursos citados por código y sílabo, y genera indicadores agregados de colocación para la Oficina de Seguimiento al Egresado (OSE), todo ello sin revelar datos personales, sin prometer empleo y sin que ningún componente automatizado tome decisiones de contratación.

El sistema se compone de cuatro portales web (estudiante, egresado, empresa aliada, OSE), una API FastAPI, un servidor MCP de solo lectura, un módulo RAG sobre el corpus institucional y un pipeline determinista de tres etapas (filtro duro → puntaje → explicación).

---

## Glossary

- **Sistema**: instancia completa de UNDC-Empleo, incluyendo API, portales web, servidor MCP, módulo RAG y pipeline determinista.
- **API**: servicio FastAPI que expone los contratos REST del sistema.
- **Servidor MCP**: servidor de solo lectura construido en la semana 4, que expone 9 herramientas y 3 recursos para acceso a datos estructurados; la única escritura permitida es registrar una postulación con consentimiento verificado.
- **Módulo RAG**: componente de recuperación aumentada que almacena fragmentos de sílabos y reglamentos en ChromaDB con embeddings de `sentence-transformers`, y devuelve citas con metadatos de documento, código de curso y sección.
- **Pipeline**: secuencia determinista de tres etapas (Filtro Duro → Puntaje → Explicación) implementada en `packages/agent/pipeline/` con funciones puras de Python, sin modelo de lenguaje en las dos primeras etapas.
- **Filtro Duro**: primera etapa del Pipeline; evalúa si el Perfil cumple cada requisito obligatorio de la Vacante; produce un resultado booleano por requisito y un indicador global de cumplimiento.
- **Puntaje**: segunda etapa del Pipeline; calcula un valor numérico entre 0 y 100 usando aritmética `Decimal` con redondeo explícito; el mismo par Perfil-Vacante produce siempre el mismo valor y la misma Huella SHA-256.
- **Huella SHA-256**: resumen criptográfico de la entrada (par Perfil-Vacante) registrado en `match_registro` junto con las tres etapas en JSONB y la versión del Pipeline.
- **LLM**: modelo de lenguaje grande (Gemini API como primario, Groq como respaldo); su único rol es redactar la Explicación sobre una estructura ya calculada por el Pipeline; nunca calcula, ordena, selecciona cursos ni recibe nombre ni código del estudiante.
- **Validador**: componente que analiza la salida textual del LLM antes de entregarla al usuario; rechaza el texto y activa la Plantilla Determinista si detecta cifras inventadas, códigos inexistentes, promesas de empleo, descarte de candidatos o mención de Atributos Protegidos.
- **Plantilla Determinista**: texto de respaldo generado sin LLM que describe el resultado del Pipeline usando solo los datos ya calculados; se activa cuando el Validador rechaza la salida del LLM o cuando no hay clave de LLM configurada.
- **Perfil**: conjunto de competencias, cursos aprobados, talleres verificados y evidencias académicas de un Estudiante o Egresado, generado sintéticamente por `data/sintetica/generador.py` con semilla fija; no contiene Atributos Protegidos.
- **Vacante**: oferta de empleo publicada por una Empresa Aliada, que declara requisitos duros (obligatorios) y deseables (opcionales).
- **Postulación**: registro de intención de un Egresado de aplicar a una Vacante, creado solo si existe Consentimiento activo; es la única operación de escritura del Servidor MCP.
- **Consentimiento**: registro versionado de la decisión del Egresado de permitir que sus datos se compartan con empresas; incluye marca de tiempo, versión y estado activo/revocado.
- **Ruta de Cierre**: lista ordenada de cursos UNDC con código y sílabo disponibles, recomendados para cerrar las brechas detectadas entre el Perfil y la Vacante; solo se genera si existe respaldo en el Servidor MCP (código) y en el Módulo RAG (sílabo).
- **Estudiante**: usuario con código universitario activo que consulta afinidad y rutas, pero no postula directamente a vacantes.
- **Egresado**: usuario con código universitario que completó sus estudios; puede postular a vacantes y gestionar su Consentimiento.
- **Empresa Aliada**: organización que publica Vacantes y consulta el Ranking de postulantes que otorgaron Consentimiento.
- **OSE**: Oficina de Seguimiento al Egresado; consulta indicadores agregados por cohorte, nunca filas individuales.
- **Ranking**: lista de postulantes a una Vacante, ordenada de mayor a menor Puntaje, que incluye a todos los postulantes que otorgaron Consentimiento; quienes no cumplen el Filtro Duro aparecen al final con el motivo explícito.
- **Atributos Protegidos**: edad, género, foto, distrito de residencia, colegio de procedencia y estado civil; ninguno se almacena como columna ni se usa en cálculos; su aparición en cualquier entrada se descarta y queda constancia en el registro.
- **Webhook de Asistencia**: llamada HTTP reproducible que registra la asistencia verificada de un Estudiante o Egresado a un taller; no requiere hardware IoT como entregable.
- **Indicadores Agregados**: métricas de colocación y participación calculadas por cohorte (año de egreso, carrera) sin exponer filas individuales.
- **JWT**: token de autenticación de corta duración emitido por la API tras un login exitoso.
- **Ley N.° 29733**: Ley de Protección de Datos Personales del Perú; rige el tratamiento de datos en el sistema.

---

## Restricciones Transversales (Reglas No Negociables)

Las siguientes restricciones aplican a todos los requisitos del sistema:

- **R1 · Nunca descarta automáticamente**: el Sistema ordena y explica; ubica al final del Ranking, con el motivo, a quien no cumple el Filtro Duro; ningún candidato es eliminado de la lista.
- **R2 · Atributos Protegidos**: los Atributos Protegidos no existen como columnas en la base de datos; si aparecen en una entrada, el Sistema los descarta y registra la ocurrencia.
- **R3 · Explicación completa**: todo resultado de afinidad nombra coincidencias, brechas y Ruta de Cierre.
- **R4 · Cita obligatoria**: toda recomendación de curso incluye código de curso y referencia al sílabo recuperado por el Módulo RAG.
- **R5 · Sin promesas**: ninguna salida del Sistema promete empleo ni colocación; la decisión de contratar corresponde exclusivamente a una persona.
- **R6 · Consentimiento**: ningún dato de un Egresado se comparte con una Empresa Aliada sin Consentimiento `compartir_con_empresas` activo; el Sistema verifica el Consentimiento al postular, al listar candidatos y al registrar la Postulación.
- **R7 · Determinismo**: el mismo par Perfil-Vacante produce siempre el mismo Puntaje y la misma Huella SHA-256, independientemente del entorno de ejecución.

---

## Requirements

### Requisito 1 — Autenticación y Sesión

**Historia de usuario:** Como usuario del sistema (Estudiante, Egresado, Empresa Aliada u OSE), quiero autenticarme con mis credenciales propias para acceder a las funciones que corresponden a mi rol.

#### Criterios de Aceptación

1. WHEN un Estudiante o Egresado envía su código universitario y contraseña correctos, THE API SHALL emitir un JWT de acceso con duración máxima de 8 horas, un JWT de refresco con duración máxima de 7 días y un campo `rol` en el payload con el valor `estudiante` o `egresado` según corresponda.
2. WHEN una Empresa Aliada o un usuario de la OSE envía su correo electrónico y contraseña correctos, THE API SHALL emitir un JWT de acceso con duración máxima de 8 horas, un JWT de refresco con duración máxima de 7 días y un campo `rol` en el payload con el valor `empresa` u `ose` según corresponda.
3. IF las credenciales enviadas no coinciden con ningún registro activo, THEN THE API SHALL responder con código HTTP 401 y un mensaje genérico que no distingue si el error es de identificador o de contraseña.
4. WHEN un cliente envía un JWT de acceso vencido junto con un JWT de refresco válido, THE API SHALL emitir un nuevo JWT de acceso con duración máxima de 8 horas sin requerir que el usuario ingrese sus credenciales nuevamente.
5. IF un cliente envía un JWT de refresco vencido o inválido, THEN THE API SHALL responder con código HTTP 401 y requerir un nuevo inicio de sesión.
6. THE API SHALL almacenar las contraseñas exclusivamente como hash bcrypt con factor de costo mínimo de 12; ninguna contraseña en texto plano se persiste en la base de datos ni aparece en registros de log.
7. WHEN un usuario autenticado intenta acceder a un recurso fuera de su rol declarado en el JWT, THE API SHALL responder con código HTTP 403 sin revelar el contenido ni la existencia del recurso.
8. IF un cliente envía una solicitud de refresco con un cuerpo malformado o sin el campo `refresh_token`, THEN THE API SHALL responder con código HTTP 422 distinguiendo este error de los fallos de autenticación.

---

### Requisito 2 — Consulta de Afinidad Perfil-Vacante

**Historia de usuario:** Como Estudiante, quiero conocer mi afinidad con una vacante concreta para saber qué requisitos acredito y cuáles me faltan antes de postular.

#### Criterios de Aceptación

1. WHEN un Estudiante autenticado solicita la afinidad entre su Perfil y una Vacante identificada por su ID, THE Sistema SHALL ejecutar el Pipeline completo (Filtro Duro → Puntaje → Explicación) y devolver el resultado en un único objeto JSON validado con Pydantic.
2. THE Pipeline SHALL calcular el Puntaje usando exclusivamente aritmética `Decimal` con redondeo a dos decimales, sin invocar el LLM en las etapas de Filtro Duro ni de Puntaje.
3. WHEN el Pipeline recibe el mismo par Perfil-Vacante en dos ejecuciones distintas, THE Sistema SHALL producir exactamente el mismo Puntaje y la misma Huella SHA-256 en ambas ejecuciones.
4. THE Sistema SHALL registrar cada ejecución del Pipeline en la tabla `match_registro` con las tres etapas en formato JSONB, la versión del Pipeline en formato MAJOR.MINOR.PATCH y la Huella SHA-256 de la entrada.
5. WHEN el LLM genera la Explicación, THE Validador SHALL analizar el texto antes de entregarlo; IF el texto contiene valores numéricos no presentes en la estructura calculada, códigos de curso no devueltos por el Servidor MCP, promesas de empleo, descarte de candidatos o mención de Atributos Protegidos, THEN THE Sistema SHALL descartar el texto del LLM, activar la Plantilla Determinista y registrar el motivo del rechazo.
6. WHERE no hay clave de LLM configurada en las variables de entorno, THE Sistema SHALL generar la Explicación usando la Plantilla Determinista sin intentar invocar ningún modelo externo.
7. THE respuesta de afinidad SHALL incluir: Puntaje entre 0.00 y 100.00, lista de coincidencias, lista de brechas, Ruta de Cierre (lista vacía si no hay respaldo), texto de Explicación y campo `explicacion_origen` con valor `"llm"` o `"plantilla"`.
8. IF la Vacante consultada no declara ningún requisito, THEN THE Sistema SHALL establecer el Puntaje como nulo, informar al Estudiante que la Vacante no tiene requisitos declarados y devolver Ruta de Cierre como lista vacía.
9. IF la Vacante identificada por el ID proporcionado no existe en el Servidor MCP, THEN THE Sistema SHALL responder con código HTTP 404 y no ejecutar el Pipeline.

---

### Requisito 3 — Ruta de Cierre de Brechas

**Historia de usuario:** Como Estudiante, quiero recibir una ruta de cursos UNDC para cerrar las brechas detectadas entre mi perfil y la vacante que me interesa.

#### Criterios de Aceptación

1. WHEN el Sistema genera una Ruta de Cierre, THE Módulo RAG SHALL recuperar el fragmento de sílabo correspondiente a cada curso recomendado y adjuntar la cita con metadatos de documento, código de curso y sección.
2. IF el Servidor MCP no encuentra el código de un curso candidato o el Módulo RAG no encuentra el sílabo correspondiente, THEN THE Sistema SHALL excluir ese curso de la Ruta de Cierre y declarar explícitamente en la respuesta que no se encontró respaldo suficiente para recomendarlo.
3. IF el Módulo RAG no devuelve ningún fragmento con puntuación de similitud coseno ≥ 0.70 para ninguna brecha, THEN THE Sistema SHALL declarar que no encontró información de cursos disponibles y devolver la Ruta de Cierre como lista vacía.
4. IF el Módulo RAG devuelve fragmentos con puntuación de similitud coseno ≥ 0.70 solo para algunas brechas y no para otras, THEN THE Sistema SHALL incluir recomendaciones únicamente para las brechas con respaldo y declarar explícitamente qué brechas quedaron sin cobertura.
5. WHEN el Sistema ordena los cursos de la Ruta de Cierre, THE Sistema SHALL usar exclusivamente la puntuación de similitud del Módulo RAG de mayor a menor; el LLM no reordena ni selecciona los cursos.
6. WHEN el Sistema devuelve un ítem de la Ruta de Cierre, THE Sistema SHALL incluir: código de curso, nombre del curso, sección del sílabo recuperada y fragmento textual de referencia de no más de 300 caracteres.

---

### Requisito 4 — Gestión de Consentimiento

**Historia de usuario:** Como Egresado, quiero decidir con quién se comparten mis datos y poder revocar ese permiso en cualquier momento para ejercer mis derechos sobre mi información personal.

#### Criterios de Aceptación

1. WHEN un Egresado autenticado activa el consentimiento `compartir_con_empresas`, THE API SHALL crear un registro de Consentimiento con marca de tiempo ISO 8601 en UTC, versión incremental y estado activo, y devolver confirmación con el identificador del registro; IF ya existe un Consentimiento activo para ese Egresado, THEN THE API SHALL devolver el identificador del registro existente sin crear un duplicado.
2. WHEN un Egresado autenticado revoca el consentimiento `compartir_con_empresas`, THE API SHALL actualizar el estado del registro de Consentimiento a revocado con marca de tiempo de revocación en UTC en un plazo máximo de 500 ms desde la recepción de la solicitud; IF no existe un Consentimiento activo para ese Egresado, THEN THE API SHALL responder con código HTTP 409 indicando que no hay consentimiento activo que revocar.
3. IF un Egresado intenta postular a una Vacante sin tener Consentimiento `compartir_con_empresas` activo, THEN THE Sistema SHALL bloquear la operación con código HTTP 403, no transmitir ningún dato del Egresado a la Empresa Aliada y devolver un mensaje que indica el tipo de consentimiento requerido y el endpoint para activarlo.
4. THE API SHALL conservar el historial completo de Consentimientos de cada Egresado durante al menos 365 días desde la fecha de cada registro, sin eliminar versiones anteriores.
5. WHEN un Egresado solicita el listado de sus Consentimientos activos e históricos (derecho de acceso ARCO), THE API SHALL devolver todos los registros del Egresado en orden cronológico descendente; IF el Egresado no tiene registros de Consentimiento, THEN THE API SHALL devolver una lista vacía con código HTTP 200.
6. WHILE el Sistema ejecuta una operación que comparte datos con Empresas Aliadas, THE Sistema SHALL verificar el estado activo del Consentimiento `compartir_con_empresas` del Egresado antes de ejecutar dicha operación.

---

### Requisito 5 — Postulación a Vacante

**Historia de usuario:** Como Egresado, quiero postular a una vacante y que mi postulación quede registrada de forma verificable.

#### Criterios de Aceptación

1. WHEN un Egresado autenticado con Consentimiento activo solicita postular a una Vacante activa, THE Servidor MCP SHALL registrar la Postulación con identificador de Egresado, identificador de Vacante, marca de tiempo ISO 8601 en UTC y Huella SHA-256 calculada sobre la concatenación canónica de `{egresado_id}|{vacante_id}|{timestamp_iso}`.
2. IF el Servidor MCP detecta que ya existe una Postulación con estado `pendiente` o `en_proceso` del mismo Egresado para la misma Vacante, THEN THE Servidor MCP SHALL rechazar la operación con código HTTP 409 y devolver el identificador de la Postulación existente.
3. IF la Vacante identificada en la solicitud de Postulación está en estado `cerrada` o `incompleta`, THEN THE Servidor MCP SHALL rechazar la operación con código HTTP 422 e indicar el estado de la Vacante.
4. THE Servidor MCP SHALL rechazar toda operación de escritura distinta al registro de Postulación con Consentimiento verificado, devolviendo código HTTP 405.
5. WHEN se registra una Postulación exitosa, THE API SHALL devolver al Egresado el identificador de Postulación, la marca de tiempo en ISO 8601 UTC y el estado inicial `pendiente`.

---

### Requisito 6 — Publicación de Vacantes por Empresa Aliada

**Historia de usuario:** Como Empresa Aliada, quiero publicar vacantes con requisitos duros y deseables para que el sistema calcule la afinidad con los perfiles de los postulantes.

#### Criterios de Aceptación

1. WHEN una Empresa Aliada autenticada envía una solicitud de creación de Vacante con al menos un requisito duro o deseable, THE API SHALL persistir la Vacante con estado `activa` y devolver su identificador único.
2. IF una Empresa Aliada envía una solicitud de creación de Vacante sin ningún requisito declarado, THEN THE API SHALL aceptar la Vacante con estado `incompleta` e incluir en la respuesta HTTP la advertencia de que el puntaje de afinidad será nulo hasta que se declaren requisitos.
3. WHEN una Empresa Aliada actualiza los requisitos de una Vacante en estado `incompleta`, THE API SHALL cambiar el estado de la Vacante a `activa` y recalcular la afinidad de los Perfiles de Egresados con Consentimiento activo que ya postularon.
4. WHEN una Empresa Aliada solicita cerrar una Vacante en estado `activa`, THE API SHALL cambiar su estado a `cerrada` y rechazar nuevas Postulaciones para esa Vacante.
5. IF una Empresa Aliada intenta actualizar o cerrar una Vacante que ya está en estado `cerrada`, THEN THE API SHALL responder con código HTTP 409 indicando que la Vacante ya está cerrada.
6. THE API SHALL validar que los campos de la Vacante no contengan Atributos Protegidos como requisitos; IF los contienen, THEN THE API SHALL rechazar la Vacante con código HTTP 422 e indicar los campos que deben corregirse.

---

### Requisito 7 — Ranking Explicado de Postulantes para Empresa Aliada

**Historia de usuario:** Como Empresa Aliada, quiero ver un ranking explicado de los postulantes a mis vacantes para tomar decisiones de selección informadas.

#### Criterios de Aceptación

1. WHEN una Empresa Aliada autenticada solicita el Ranking de una Vacante, THE Sistema SHALL listar únicamente los Egresados que postularon con Consentimiento `compartir_con_empresas` activo al momento de la consulta.
2. THE Sistema SHALL ordenar el Ranking de mayor a menor Puntaje e incluir a todos los postulantes con Consentimiento activo, sin eliminar ninguno de la lista.
3. IF un Egresado no cumple el Filtro Duro de la Vacante, THEN THE Sistema SHALL ubicarlo al final del Ranking con el motivo explícito de incumplimiento por cada requisito duro no satisfecho.
4. THE Sistema SHALL incluir en cada entrada del Ranking: Puntaje, coincidencias, brechas, Ruta de Cierre resumida de máximo 3 ítems y texto de Explicación; ningún campo del Ranking contendrá Atributos Protegidos.
5. IF el Consentimiento de un Egresado es revocado antes de que la Empresa Aliada consulte el Ranking, THEN THE Sistema SHALL excluir al Egresado de la respuesta sin indicar su identidad ni su existencia previa en la lista.
6. IF la Vacante no tiene postulantes con Consentimiento activo, THEN THE Sistema SHALL devolver una lista de Ranking vacía con código HTTP 200 y una nota indicando que no hay postulantes disponibles.
7. THE Sistema SHALL acompañar cada entrada del Ranking con el aviso: "La decisión de contratación corresponde exclusivamente a una persona; este sistema no descarta candidatos."

---

### Requisito 8 — Indicadores Agregados para la OSE

**Historia de usuario:** Como responsable de la OSE, quiero consultar indicadores de colocación y participación por cohorte para medir la empleabilidad de los egresados con evidencia verificable.

#### Criterios de Aceptación

1. WHEN un usuario de la OSE autenticado consulta los indicadores, THE API SHALL devolver exclusivamente cifras agregadas por cohorte (año de egreso y carrera), sin exponer ninguna fila individual de Egresado.
2. THE API SHALL calcular los Indicadores Agregados con las siguientes métricas por cohorte: número de egresados activos en el sistema, número de Postulaciones registradas, tasa de Postulaciones por egresado, número de Consentimientos activos y número de talleres verificados.
3. WHEN los Indicadores Agregados provienen de datos sintéticos generados con semilla fija, THE API SHALL incluir en la respuesta el aviso: "Los indicadores se calculan sobre datos sintéticos generados con semilla fija; la correlación observada fue sembrada en el generador y no refleja datos reales."
4. IF el número de Egresados en una cohorte es menor a 5, THEN THE API SHALL omitir los indicadores de esa cohorte y notificar que el grupo es demasiado pequeño para publicar cifras sin riesgo de identificación.
5. IF la solicitud de la OSE incluye como parámetro de filtro el identificador de Egresado, nombre, código universitario, correo electrónico o cualquier combinación de estos atributos, THEN THE API SHALL responder con código HTTP 422 indicando que los filtros de identificación individual no están permitidos.

---

### Requisito 9 — Webhook de Asistencia Verificada a Talleres

**Historia de usuario:** Como Estudiante o Egresado, quiero que mi asistencia verificada a un taller sea reconocida como evidencia de competencia para mejorar mi puntaje de afinidad.

#### Criterios de Aceptación

1. WHEN el Sistema recibe una llamada al Webhook de Asistencia con identificador de usuario, identificador de taller y firma HMAC-SHA256 válida calculada sobre `{usuario_id}|{taller_id}|{timestamp_iso}` usando la clave `WEBHOOK_SECRET`, THE API SHALL registrar la asistencia con marca de tiempo ISO 8601 en UTC y marcar la competencia asociada al taller como `acreditada_por_asistencia` en el Perfil del usuario.
2. WHEN se registra una asistencia verificada, THE Sistema SHALL recalcular el Puntaje de todas las afinidades previas del usuario que incluyan la competencia acreditada, usando el mismo Pipeline determinista.
3. THE Sistema SHALL registrar la asistencia verificada exclusivamente en la tabla `taller_evidencia`; la tabla de historia académica y el CV del usuario no se modifican bajo ninguna circunstancia.
4. IF la firma HMAC-SHA256 del Webhook no coincide con la calculada por el sistema, THEN THE API SHALL responder con código HTTP 401 y registrar el intento fallido en la tabla de auditoría con marca de tiempo, identificador de taller recibido y motivo del rechazo.
5. IF el identificador de taller no existe en el catálogo de talleres, THEN THE API SHALL responder con código HTTP 404 y registrar el intento fallido en la tabla de auditoría.
6. THE Webhook de Asistencia SHALL ser invocable mediante una llamada HTTP POST directa sin necesidad de hardware IoT; el sistema acepta la llamada HTTP como único mecanismo de registro.

---

### Requisito 10 — Servidor MCP y Acceso a Datos Estructurados

**Historia de usuario:** Como componente interno del sistema, el Pipeline y la API necesitan acceder a datos estructurados de forma consistente a través del Servidor MCP.

#### Criterios de Aceptación

1. THE Servidor MCP SHALL exponer exactamente 9 herramientas y 3 recursos de solo lectura sobre la base de datos estructurada.
2. WHEN el Pipeline solicita datos de Perfil o Vacante al Servidor MCP con un máximo de 10 solicitudes concurrentes en entorno de desarrollo, THE Servidor MCP SHALL devolver la respuesta en menos de 2 segundos.
3. THE Servidor MCP SHALL rechazar toda operación de escritura excepto el registro de Postulación con Consentimiento verificado, devolviendo código HTTP 405.
4. WHEN el Servidor MCP recibe una solicitud de registro de Postulación, THE Servidor MCP SHALL verificar la existencia y estado activo del Consentimiento `compartir_con_empresas` del Egresado antes de persistir el registro; IF el Consentimiento no existe o no está activo, THEN THE Servidor MCP SHALL rechazar la operación con código HTTP 403.
5. THE API SHALL producir los mismos contratos de datos que el Servidor MCP para las operaciones de afinidad, Ranking e Indicadores; el script `verificar_determinismo.py` SHALL confirmar la misma Huella SHA-256 y el mismo Puntaje calculados por ambos caminos.

---

### Requisito 11 — Módulo RAG sobre Corpus Institucional

**Historia de usuario:** Como componente interno del sistema, la generación de Rutas de Cierre necesita recuperar fragmentos relevantes de sílabos y reglamentos con citas verificables.

#### Criterios de Aceptación

1. THE Módulo RAG SHALL indexar los sílabos y reglamentos del corpus institucional en ChromaDB usando embeddings generados con `sentence-transformers` multilingüe; cada fragmento indexado SHALL conservar metadatos de documento, código de curso y sección.
2. WHEN el Sistema solicita fragmentos para una brecha específica, THE Módulo RAG SHALL devolver los fragmentos ordenados por puntuación de similitud coseno descendente, junto con sus metadatos completos.
3. IF el Módulo RAG no encuentra ningún fragmento con puntuación de similitud coseno ≥ 0.50 (umbral por defecto configurable), THEN THE Sistema SHALL declarar explícitamente que no encontró respaldo para esa brecha y no incluir ninguna recomendación de curso para ella.
4. THE Módulo RAG SHALL funcionar exclusivamente con ejecución local; no realizará llamadas a servicios externos de embeddings durante la operación normal.

---

### Requisito 12 — Privacidad, Seguridad y Cumplimiento

**Historia de usuario:** Como sistema que trata datos de personas, el Sistema debe cumplir con la Ley N.° 29733 y proteger la privacidad de los usuarios.

#### Criterios de Aceptación

1. THE Sistema SHALL operar exclusivamente con datos sintéticos generados por `data/sintetica/generador.py` con semilla fija durante el desarrollo; ningún dato personal real se almacena ni procesa.
2. THE API SHALL inspeccionar toda entrada recibida en busca de Atributos Protegidos (edad, género, foto, distrito, colegio, estado civil); IF se detectan, THEN THE API SHALL descartarlos antes de procesarlos.
3. WHEN la API descarta Atributos Protegidos de una entrada, THE API SHALL registrar la ocurrencia con marca de tiempo ISO 8601 en UTC, el tipo de atributo descartado y el endpoint involucrado, y continuar el procesamiento sin esos campos.
4. THE Sistema SHALL transmitir las claves `GEMINI_API_KEY` y `GROQ_API_KEY` exclusivamente mediante variables de entorno; ninguna clave SHALL aparecer en el código fuente ni en el repositorio.
5. WHEN el LLM es invocado, THE Sistema SHALL omitir el nombre y el código universitario del Estudiante o Egresado de toda entrada enviada al modelo externo.
6. THE Sistema SHALL registrar en `match_registro` cada ejecución del Pipeline con las tres etapas en JSONB, la versión del Pipeline en formato MAJOR.MINOR.PATCH, la Huella SHA-256 y la marca de tiempo ISO 8601 en UTC; este registro es inmutable una vez creado.
