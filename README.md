# Buenas prácticas de desarrollo

_Oracle Database, Oracle APEX y desarrollo asistido por IA_

Esta guía reúne las prácticas que hemos adoptado al desarrollar aplicaciones con Oracle APEX y Oracle Database trabajando junto a un asistente de IA. Está pensada como referencia para el equipo: cada práctica explica **qué** hacer y **por qué**.

## Contenido

- [0. Arranque](#0-arranque)
- [1. Control de versiones y entornos](#1-control-de-versiones-y-entornos)
- [2. Trabajo con IA y acceso a la base de datos](#2-trabajo-con-ia-y-acceso-a-la-base-de-datos)
- [3. Skills y herramientas](#3-skills-y-herramientas)
- [4. Preferir los componentes nativos de APEX](#4-preferir-los-componentes-nativos-de-apex)
- [5. Gestión del contexto de la IA](#5-gestión-del-contexto-de-la-ia)
- [6. Elegir el modelo de Claude según la tarea](#6-elegir-el-modelo-de-claude-según-la-tarea)
- [7. Documentación y rigor técnico](#7-documentación-y-rigor-técnico)
- [8. Cierre de tarea y limpieza de archivos sobrantes](#8-cierre-de-tarea-y-limpieza-de-archivos-sobrantes)
- [9. Lista de verificación](#9-lista-de-verificación)

---

## 0. Arranque

### 0.1 Descargar Claude Desktop

Claude Code está optimizado para que se inicien los trabajos con Claude Desktop. Se descarga tanto desde la página de Anthropic como desde la tienda de Microsoft.

### 0.2 Configurar Claude Desktop

En la Configuración (se accede presionando sobre nuestro nombre, abajo a la derecha) podemos ajustar la herramienta según nuestro gusto.

- **Es fundamental** que en Configuración → Privacidad → *Help improve our AI models* (Ayúdanos a mejorar nuestros modelos de IA, o traducción similar) el indicador esté **APAGADO**. Así se evita que Anthropic use nuestra información.
- Configuración → Uso → **Sesión actual**: da 5 horas de uso. Si llegamos a ese límite, tenemos que esperar a que esas horas terminen.
- Configuración → Uso → **Esta semana**: es el segundo límite de Claude; todas las semanas se renuevan los límites y vuelven a 0 %.
- Con el tiempo pueden aparecer filas de «Restablecimiento completo», que permiten restablecer ambos límites a 0 % en cualquier momento. Claude suele regalar esos restablecimientos.

### 0.3 Verificar skill

Para el uso de APEX y de la base de datos, hay que preguntarle a la IA si ya está disponible el skill de Oracle para APEX y Base de Datos. El skill ayuda a que la IA sepa cómo se utiliza nuestra herramienta de desarrollo.

### 0.4 SQLcl y Java

La IA tendrá acceso a las funciones de APEXlang y a la base de datos en general mediante SQLcl, la herramienta que viene a sustituir al antiguo SQL*Plus.

Para usar SQLcl es necesario tener Java 17 o superior.

## 1. Control de versiones y entornos

### 1.1 Versionar todo en GitHub

Todo proyecto vive en un repositorio Git con remoto en GitHub (cada gerencia de la empresa tiene su propia organización): el código de la aplicación (exportado como APEXlang), los scripts de base de datos y la documentación.

- **Nunca versionar secretos**: wallets, certificados (`.p12`, `.pem`), archivos `.env`, credenciales y configuraciones de clientes. Dejarlos en `.gitignore`; si hace falta una plantilla, versionar un archivo `*.example` sin valores reales.
- **Nunca versionar datos reales**: dumps, exports con datos de usuarios, hashes de contraseñas o movimientos.
- Hacer commits pequeños y con mensajes que expliquen el motivo del cambio, no solo qué archivos se tocaron.

### 1.2 Una única fuente de código para varias aplicaciones

El código exportado de la aplicación en producción es la única fuente. A partir de esa misma carpeta se generan la aplicación de producción y una o más aplicaciones de prueba; la aplicación de prueba no tiene código propio.

- Todo cambio grande o si hay creación de páginas nuevas se prueba primero en la aplicación de prueba.
- Mantener los cambios sincronizados con GitHub.
- Los scripts de importación fijan siempre workspace, ID, nombre y alias de la aplicación destino, para que cada una conserve su identidad sin importar desde dónde se exportó el código. Hay que tener cuidado con eso. También debe usarse **Thin** en la conexión con SQLcl.
- Si se hacen cambios en el App Builder de la aplicación de prueba, se bajan al repositorio con un export y se revisan con `git diff` antes de hacer commit. Las diferencias esperables (nombre, alias, ID) no se commitean.

### 1.3 Working copies de APEX

Las working copies permiten trabajar un cambio o una funcionalidad aislada de la aplicación principal y fusionarla cuando está probada. Funcionan como las ramas de Git, pero dentro de APEX.

- Una working copy por cambio o funcionalidad, con un nombre que describa su propósito.
- Fusionar a la aplicación principal solo después de probar, y descartar la working copy una vez fusionada.
- No acumular working copies abiertas: cada una que sigue viva se aleja de la principal y hace más difícil la fusión.

### 1.4 Exportar y hacer commit antes de importar

`apex import` reemplaza la aplicación completa, sin fusionar con lo que haya en la instancia. Por eso el ciclo es siempre: **exportar → hacer commit → editar → validar → importar**.

- Exportar primero aunque nadie haya entrado al App Builder: el uso normal de la aplicación también deja huella (por ejemplo, los reportes interactivos guardados por los usuarios).
- Avisar antes de importar si hay personas usando la aplicación: el import borra sus reportes privados y preferencias.
- Para saber qué cambió realmente en una aplicación, mirar `LAST_UPDATED_BY` y `LAST_UPDATED_ON` en las vistas `APEX_APPLICATION_PAGE_*`. El diff de archivos no sirve porque cada export normaliza el formato.

### 1.5 Separar los entornos

- Cada entorno (nube, servidor propio, pruebas) tiene su propia conexión SQLcl con nombre y sufijo identificable.
- Los IDs de aplicación, schemas y workspaces son distintos entre entornos: un import cruzado pisa una aplicación real. Prohibir copiar archivos `.apx` entre proyectos sin instrucción expresa.
- Ante cualquier duda sobre a qué entorno pertenece algo, preguntar antes de actuar.

## 2. Trabajo con IA y acceso a la base de datos

### 2.1 Acceso mediante SQLcl con conexiones guardadas

Dar a la IA acceso a la base a través de SQLcl le permite consultar el diccionario, ejecutar scripts, exportar, validar e importar aplicaciones y comprobar el resultado sin intermediarios.

- Usar **conexiones guardadas con nombre** (`sql -name <conexión>`), de modo que no hagan falta credenciales en los comandos ni en los archivos del proyecto.
- Aplicar el **mínimo privilegio**: para el desarrollo diario conviene un usuario del schema de la aplicación, o de solo lectura para investigar, y no `ADMIN`. Reservar los privilegios altos para tareas puntuales (otorgar un grant, crear un schema).
- Verificar con consultas en lugar de suponer. Las vistas del diccionario de APEX (`apex_debug_messages`, `apex_workspace_activity_log`, `apex_application_page_*`) resuelven en minutos problemas que de otro modo llevan horas de prueba y error.

### 2.2 Reglas escritas para la IA

Las reglas se escriben en un archivo `CLAUDE.md` en la raíz del proyecto, que la IA carga antes de actuar. Debe incluir:

- Qué entorno, base, workspace, schema y aplicaciones son los de este proyecto, y cuáles no se tocan.
- Qué acciones requieren aprobación explícita: `apex import`, borrados masivos, scripts de reseteo, cambios de privilegios. Si se concede una autorización permanente, dejarla escrita con fecha y alcance.
- Prohibido inventar nombres de columnas, vistas o propiedades de APEX que no estén confirmados contra la base o contra el compilador.

> **Principio**
>
> Toda acción destructiva o irreversible se confirma con una persona. La IA propone y ejecuta lo seguro; lo que pisa o borra, se aprueba.

### 2.3 Scripts de base de datos seguros

- Numerar los scripts (`00_`, `01_`, …) y hacerlos **re-ejecutables** o con control de «ya aplicado». Los scripts reciben el schema como parámetro.
- Ejecutarlos siempre con `@archivo`, nunca por la entrada estándar: por pipe la codificación puede corromper caracteres como la «ñ» sin dar ningún error.
- Los scripts que borran de varias tablas en una transacción empiezan con `ALTER SESSION DISABLE PARALLEL DML`, porque Autonomous Database habilita DML paralelo por defecto y falla con ORA-12839.
- Separar las migraciones de estructura y datos (se corren una vez, en orden) del código de la base (paquetes, vistas, triggers), que se vuelve a aplicar completo.

### 2.4 SSH (avanzado)

También se puede optar por habilitar conexiones con claves para usar SSH con los servidores del cliente, algo muy útil para modificar archivos con JASPER, ver problemas en los servidores, etc.

A la vez, esto es peligroso porque la IA estaría accediendo a **TODO** el servidor del cliente, por lo que antes debemos darle normas y tareas bien claras para evitar problemas.

## 3. Skills y herramientas

### 3.1 Usar el skill oficial de Oracle

Oracle publica un skill para generar y validar APEXlang. Consultarlo antes de editar evita propiedades inválidas y agiliza el ciclo.

- Decidir una sola política para el skill y documentarla: o se versiona dentro del repositorio (todos trabajan con la misma versión), o se clona fuera y se ignora en Git (se evita duplicar código externo).
- El skill y el compilador pueden diferir de lo que el validador acepta. **La autoridad final es** `apex validate`: probar una página antes de replicar el patrón en veinte.

### 3.2 Complementar la validación con herramientas propias

La validación fuera de línea no detecta todo lo que depende del render. Conviene escribir pequeñas verificaciones para los errores que ya ocurrieron.

- Un chequeo de la grilla de 12 columnas: `apex validate` e `import` aceptan una fila con más de 12, pero la página falla al renderizar y devuelve HTTP 400.
- Generadores para el código repetitivo (páginas con la misma estructura), en lugar de editar decenas de archivos a mano.
- Un `RUNBOOK` con los comandos del día a día, para no depender de la memoria de nadie.

## 4. Preferir los componentes nativos de APEX

Los asistentes de IA tienden a resolver cualquier necesidad de interfaz generando HTML y CSS a mano. Funciona, pero cuesta mantenerlo. **La regla es usar primero lo que APEX ya ofrece** y recurrir a HTML y CSS propios solo cuando no exista una alternativa nativa.

### 4.1 Por qué importa

- **Mantenimiento y actualizaciones**: los componentes nativos evolucionan con APEX y el Universal Theme; el HTML y el CSS a mano se rompen con los cambios de versión.
- **Funciones gratis**: un reporte interactivo trae búsqueda, filtros, orden, quiebres y descarga. Una tabla armada con HTML desde PL/SQL pierde ordenamiento, paginación y descarga, y hay que reconstruirlos.
- **Seguridad**: el HTML generado en PL/SQL obliga a escapar cada valor con `APEX_ESCAPE.HTML`; un descuido es una inyección de código. Los componentes nativos escapan por defecto.
- **Accesibilidad y diseño adaptable**: los componentes del tema ya cumplen con ambas; el HTML propio hay que probarlo.
- **Menos código**: lo declarativo ocupa menos y se revisa más fácilmente en el diff.

### 4.2 Qué usar en cada caso

| Necesidad | Opción nativa a probar primero |
|---|---|
| Listados de datos | Reporte interactivo (pantalla principal) o reporte clásico (apoyo junto a un formulario). |
| Listas de valores | LOV compartidas, en lugar de campos donde se escriben códigos o IDs. |
| Distribución de campos | Plantillas de región y de item, grilla de 12 columnas y opciones de plantilla (template options) del tema. |
| Colores y apariencia | Theme Styles / Theme Roller y variables CSS del Universal Theme (`--ut-*`). |
| Comportamiento en pantalla | Dynamic Actions antes que JavaScript propio. |
| Permisos | Authorization Schemes en página y en proceso, no solo ocultar botones. |
| Mensajes y validaciones | Validaciones y procesos declarativos con mensajes de APEX. |
| Íconos | Font APEX incluido en el tema. |

### 4.3 Cuando no hay alternativa nativa

- Concentrar el CSS en un único archivo de la aplicación (Static Application Files), cargado después del tema, sin `!important`.
- Sobrescribir las variables del tema en lugar de pelear contra sus selectores.
- No usar estilos en línea para decisiones de diseño. El estilo en línea es aceptable solo para transportar un **dato** (por ejemplo, el color real de un registro).
- No cargar recursos externos (CDN, fuentes web, librerías de íconos): si el recurso no carga, el diseño se rompe. Todo asset debe venir de los archivos de la aplicación.
- No dejar presentación dentro del PL/SQL: las funciones devuelven datos o clases semánticas; los colores y tamaños viven en el CSS.
- Dejar escrita la justificación de cada excepción en la documentación de decisiones.

> **Instrucción para la IA**
>
> Agregar al `CLAUDE.md`: «Antes de escribir HTML o CSS propio, comprobar si existe un componente, plantilla u opción de tema nativa de APEX. Si se usa HTML o CSS propio, explicar por qué no alcanzó lo nativo.» Y al revisar propuestas de la IA, preguntar siempre: ¿existe un componente nativo para esto?

## 5. Gestión del contexto de la IA

La calidad de las respuestas depende de lo que la IA tiene presente en la conversación (su «contexto»). Una conversación demasiado larga o mezclada con varios temas degrada los resultados y consume más rápido los límites de uso.

### 5.1 Memoria permanente en el repositorio

- `CLAUDE.md`: lo que toda sesión nueva debe saber (entornos, convenciones, reglas, qué no tocar). Mantenerlo corto y actualizado.
- Documentación de decisiones: lo decidido queda escrito con sus alternativas, así una conversación nueva retoma sin repetir el razonamiento.
- Bitácora de hitos y `RUNBOOK`: qué se hizo, en qué estado quedó y cómo se opera.

### 5.2 Un tema por conversación

- **Mantener cada conversación enfocada en un solo tema o tarea** (por ejemplo, «la página de alta de clientes» o «la migración del módulo de facturación»). Con un contexto limpio y relevante, la IA es más efectiva y comete menos errores.
- Evitar mezclar temas que no se relacionan (corregir un error de una página, rediseñar una tabla y redactar documentación en la misma conversación): los detalles de uno estorban al otro.
- Cuando surge un tema nuevo, no seguir en la misma conversación: bifurcar (5.4) o abrir una nueva (5.6).
- **Si se quiere aprovechar el contexto de una conversación para hacer otra cosa** (la IA ya conoce el esquema, las tablas y las decisiones), no continuar en la misma: **bifurcarla**. La original queda intacta con su tema y la bifurcación arranca con todo ese contexto.

### 5.3 Compactar

Compactar resume la conversación para liberar espacio, conservando lo esencial, y permite seguir sin empezar de cero.

- Compactar cuando la conversación crece, se vuelve lenta o empieza a perder precisión, pero sigue siendo del mismo tema.
- Hacerlo en un punto de corte natural (al terminar una página, después de un commit) y no en medio de una tarea a medias.
- Indicar qué debe conservarse: decisiones tomadas, archivos modificados, pendientes y errores ya descartados. En Claude Code, `/compact` admite instrucciones sobre qué mantener.
- Un resumen siempre pierde detalle. Lo que debe sobrevivir (reglas, decisiones, nombres de objetos) tiene que estar en el `CLAUDE.md` o en la documentación, no solo en la conversación.
- Si después de compactar la IA «olvida» algo importante, es señal de que ese dato debía estar escrito en el repositorio.

### 5.4 Bifurcar (fork)

Bifurcar crea una copia de la conversación hasta ese punto, que sigue por un camino independiente. La original no se modifica.

- Para **explorar una alternativa** sin contaminar la línea principal (por ejemplo, reporte interactivo contra reporte clásico).
- Para **tareas independientes que parten del mismo contexto**: tras analizar el modelo de datos, una bifurcación hace la página y otra el script de base de datos.
- Para **reutilizar el contexto en otro tema** (ver 5.2).
- Para **probar la misma tarea con otro modelo** y comparar resultados (ver sección 6).
- Antes de un cambio arriesgado, para poder volver al punto anterior.
- Si la conversación de origen ya es muy larga, conviene compactarla antes de bifurcar para que la copia no arrastre contexto innecesario.
- Lo que se decide en una bifurcación se lleva al repositorio (commit, documentación); no se copia a mano entre conversaciones.

### 5.5 Nombrar las conversaciones y las bifurcaciones

- **Cambiar el título de cada bifurcación apenas se crea.** Si no, queda con el mismo título que la original o con uno genérico, y después nadie sabe qué hace cada una.
- Usar una convención simple, del estilo `Área · Objeto · Acción`:
  - `APEX · Pág. 12 alta de clientes · formulario`
  - `BD · Migración 004 · índices de facturas`
  - `FORK · Reporte interactivo vs. clásico (pág. 20)`
- Si el objetivo de una conversación cambia, actualizar el título (o, mejor, bifurcar según 5.2).
- Un buen título permite retomar el trabajo, revisar el historial y que otra persona entienda qué se hizo en cada conversación.
- Al terminar, el título debería describir lo que quedó hecho.

### 5.6 Empezar una conversación nueva

- Al cambiar de tarea cuando no hace falta el contexto anterior. Si lo importante quedó en el `CLAUDE.md` y en la documentación, no se pierde nada.
- Antes de cerrar una sesión larga, pedir a la IA que resuma el estado y las decisiones pendientes, y guardar ese resumen en la documentación.

### 5.7 Delegar la exploración

- Las búsquedas amplias (por ejemplo, localizar todas las páginas que usan cierto componente) se delegan a subagentes, para no llenar la conversación principal con resultados intermedios.

### 5.8 Qué hacer en cada situación

| Situación | Qué hacer |
|---|---|
| La conversación es larga, pero sigue siendo del mismo tema | Compactar |
| Quiero usar lo que la IA ya sabe para otra tarea | Bifurcar y renombrar |
| Quiero probar una alternativa sin riesgo | Bifurcar y renombrar |
| Cambio total de tema y no necesito el contexto | Conversación nueva |
| Terminé la tarea | Dejar un título que diga qué se hizo y guardar lo importante en el repositorio |

## 6. Elegir el modelo de Claude según la tarea

Claude se ofrece como una familia de tres modelos: **Opus**, **Sonnet** y **Haiku**. Las versiones actuales son las 5.5, lanzadas entre septiembre y octubre de 2026. Los tres aceptan texto e imágenes, usan herramientas, tienen una ventana de contexto de 1 millón de tokens, pueden generar hasta 128 mil tokens de salida y tienen conocimiento hasta junio de 2026. Se diferencian en **capacidad, velocidad y costo**.

### 6.1 Comparativa

| | **Opus 5.5** | **Sonnet 5.5** | **Haiku 5.5** |
|---|---|---|---|
| Para qué lo describe Anthropic | Programación agéntica de larga duración y trabajo de conocimiento | La mejor combinación de velocidad e inteligencia | Tareas de alto volumen y baja latencia: clasificación, extracción, enrutamiento |
| Capacidad | La más alta de los tres | Alta; equilibrio entre capacidad y velocidad | Buena en tareas acotadas; la menor de los tres |
| Velocidad | Moderada | Rápida | La más rápida |
| Razonamiento | Adaptativo, siempre activo (esfuerzo por defecto: medio) | Adaptativo (esfuerzo por defecto: alto); permite reducirlo | Adaptativo (esfuerzo por defecto: medio) |
| Precio por millón de tokens, entrada / salida (USD) | 4 / 20 | 2 / 10 | 0,10 / 0,50 (hasta 100 mil tokens de prompt) |
| Con prompts de más de 100 mil tokens | Sin recargo | Sin recargo | 0,50 / 2,50 |
| Lectura de caché (USD por millón) | 0,20 | 0,10 | 0,01 |
| Procesamiento por lotes (Batch API) | 50 % de descuento | 50 % de descuento | 50 % de descuento |
| ID de modelo | `claude-opus-5-5` | `claude-sonnet-5-5` | `claude-haiku-5-5` |

**Cómo leer la tabla**

- **Razonamiento adaptativo** significa que el modelo decide cuánto «pensar» antes de responder, según la dificultad. El parámetro de **esfuerzo** (effort) permite subir o bajar esa profundidad: más esfuerzo da mejores resultados en problemas difíciles, pero cuesta más y demora más. Anthropic recomienda ajustar el esfuerzo antes de cambiar de modelo.
- **Costo relativo**: Sonnet cuesta la mitad que Opus por token, y Haiku cuesta unas 40 veces menos que Opus mientras el prompt no pase de 100 mil tokens. Los tokens de salida (lo que la IA escribe) cuestan 5 veces más que los de entrada en los tres modelos.
- **Capacidad**: Anthropic no publica una única medida que ordene a los tres; el orden de la tabla sale de cómo los posiciona (Opus como el más capaz para trabajo complejo y largo, Haiku como el más rápido y económico) y es orientativo. Para nuestro caso de uso conviene comprobarlo con tareas reales de APEX y Oracle Database.
- Los precios son los de la **API**. Con un plan de Claude Desktop o Claude Code el consumo se mide en los límites de uso (ver 0.2) y no en dólares; la proporción entre modelos sirve como referencia, pero conviene confirmarla en Configuración → Uso.

> **Nota**
>
> Existe un nivel por encima de los tres: **Fable 5.1** (USD 10 / 50 por millón de tokens, más lento), pensado para razonamiento muy exigente y trabajo autónomo de muchas horas. Anthropic recomienda empezar por Opus 5.5 y pasar a Fable solo si las pruebas no alcanzan.

### 6.2 Escenarios de uso con Oracle Database y APEX

Esta guía es una recomendación del equipo para nuestro caso de uso; no proviene de pruebas publicadas por Anthropic.

**Opus 5.5: diseño, riesgo y problemas difíciles**

- Diseñar el modelo de datos, las reglas de integridad, los índices y la estructura de paquetes PL/SQL.
- Migraciones de estructura y datos con riesgo, y refactorizaciones grandes de paquetes.
- Depuración difícil: fallas silenciosas de renderizado en APEX, problemas de rendimiento, planes de ejecución, bloqueos (locks).
- Revisión de seguridad y de código antes de pasar a producción.
- Tareas largas y autónomas: construir un módulo completo con varias páginas, con exportación, validación e importación incluidas.
- Decisiones de arquitectura que van a la documentación de decisiones.

**Sonnet 5.5: el modelo para el trabajo diario**

- Crear y modificar páginas, regiones, items y procesos en APEXlang.
- Escribir consultas, vistas y paquetes PL/SQL de complejidad media.
- Corregir errores ya diagnosticados, escribir scripts SQL y pruebas.
- Documentación técnica y análisis de datos.
- El ciclo diario con SQLcl: exportar, editar, validar e importar.

**Haiku 5.5: tareas mecánicas, repetitivas y de búsqueda**

- Renombrar, formatear y aplicar el mismo cambio a muchos archivos.
- Generar páginas con una estructura idéntica a partir de una plantilla ya validada.
- Traducir o redactar mensajes, etiquetas, comentarios y mensajes de commit.
- Cargar catálogos y listas de valores con datos ya definidos.
- Búsquedas y clasificación: localizar en qué páginas se usa un componente, listar objetos del diccionario, resumir registros (logs).
- Subagentes de exploración (ver 5.7).
- **Límite**: no usarlo, sin revisión, para decisiones de diseño, de seguridad ni para cambios destructivos.

### 6.3 Reglas prácticas

1. **Empezar con Sonnet** para el trabajo diario.
2. **Subir a Opus** cuando la tarea es compleja o riesgosa (diseño, migraciones, seguridad), o cuando Sonnet se atasca después de dos intentos.
3. **Bajar a Haiku** cuando la tarea es mecánica y fácil de verificar.
4. **Ajustar el esfuerzo antes de cambiar de modelo**: bajarlo en tareas simples y subirlo en las difíciles.
5. **Probar un caso real antes de generalizar**: para una tarea repetitiva, validar el resultado con un modelo y recién entonces delegar el resto a uno más económico.
6. Cambiar de modelo en medio de una conversación larga obliga a reprocesar todo el contexto sin aprovechar la memoria caché. Si hace falta otro modelo para una tarea distinta, es mejor **bifurcar** (5.4) y cambiar el modelo en la bifurcación.
7. Revisar siempre lo que produce un modelo más económico antes de importarlo: un error barato de generar puede salir caro en producción.

### 6.4 Fuentes

Datos de la documentación oficial de Anthropic, consultada el 21/10/2026. Los precios pueden cambiar, por lo que conviene verificarlos antes de tomar decisiones de presupuesto.

- [Descripción general de los modelos](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Precios](https://platform.claude.com/docs/en/about-claude/pricing)
- [Cómo elegir un modelo](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
- [Claude Opus 5.5](https://platform.claude.com/docs/en/models/opus-5-5/overview), [Claude Sonnet 5.5](https://platform.claude.com/docs/en/models/sonnet-5-5/overview) y [Claude Haiku 5.5](https://platform.claude.com/docs/en/models/haiku-5-5/overview)

## 7. Documentación y rigor técnico

### 7.1 Registrar decisiones

- Documentar una decisión solo cuando hubo más de un camino posible y la elección condiciona el trabajo futuro.
- Cada registro lleva: contexto, alternativas evaluadas, decisión, consecuencias y estado (propuesta, aceptada, superada).
- Cuando una decisión resulta ser incorrecta, no se borra: se marca como superada y se enlaza a la que la reemplaza.
- Incluir la consulta SQL que permite verificar la decisión.

### 7.2 Verificar en el producto real

- Una validación correcta no garantiza que la pantalla se vea bien: abrir la aplicación y comprobarlo.
- Los fallos silenciosos son los más caros (un slot de región inválido, un item sin plantilla de label, un branch que hace rollback). Documentarlos con su síntoma y su causa.
- Probar un caso en una página antes de replicar el patrón en veinte.

### 7.3 Modelado de datos

- Garantizar las reglas de negocio en la base: índices únicos parciales, CHECK, claves foráneas. La aplicación no es la última línea de defensa.
- Reemplazar el texto libre por catálogos cuando el dato se usa para filtrar o agrupar.
- Definir la identidad de una entidad una sola vez (por ejemplo, en una columna virtual) y reutilizarla en vistas y paquetes.
- Las páginas consultan **vistas**, nunca tablas base, y la lógica vive en paquetes PL/SQL.
- Ante un posible falso positivo, preferir el falso negativo: un aviso equivocado cuesta la confianza en todos los demás.
- Cuidado con `ON DELETE CASCADE`: puede eliminar datos relacionados en silencio. Si la regla de negocio lo prohíbe, controlarlo explícitamente.

### 7.4 Seguridad

- Las contraseñas nunca se guardan en claro, ni se escriben en logs ni pasan por el estado de sesión: salt por usuario y hash iterado.
- La autorización se valida en la página y en el proceso, no solo en el botón.
- Los authorization schemes de rol se cachean una vez por vista de página (Once per page view), no por sesión; así quitar un rol tiene efecto inmediato.
- Activar Session State Protection en toda página que reciba un identificador por URL.
- Registrar en una tabla de auditoría de solo inserción toda acción sobre cuentas ajenas.

## 8. Cierre de tarea y limpieza de archivos sobrantes

Trabajar con IA genera muchos archivos intermedios: registros de ejecución, exportaciones de prueba, planes temporales, informes de revisión y borradores. Al terminar una tarea, esos restos no aportan nada y ensucian el repositorio, el contexto de la IA y las búsquedas.

### 8.1 Qué suele sobrar

- Archivos de registro (`*.log`) y salidas de consola guardadas.
- Documentos Markdown de trabajo que ya cumplieron su función: planes temporales, informes de sesión, críticas o revisiones puntuales, notas de depuración.
- Exportaciones de prueba y copias de respaldo antiguas de aplicaciones, cuando ya existe el commit equivalente en Git.
- Renders, capturas e imágenes intermedias; carpetas temporales (`tmp`, `__pycache__`).
- Scripts de un solo uso que ya se ejecutaron y no se volverán a necesitar.
- Duplicados: versiones `v1`, `v2`, `(1)` de un mismo archivo.

### 8.2 Qué se conserva

- `README`, `CLAUDE.md`, la documentación de decisiones, la bitácora de hitos y el `RUNBOOK`.
- Migraciones y scripts reutilizables de base de datos.
- Las herramientas de verificación y los generadores que se siguen usando.
- Los respaldos que son la única copia de algo (por ejemplo, un dato previo a una migración).

### 8.3 Cómo hacerlo

- Al cerrar la tarea, **pedir a la IA que proponga una lista de candidatos a eliminar**, con el motivo de cada uno. No debe borrar por su cuenta.
- Una persona revisa la lista y confirma. Para cada archivo, preguntar: ¿alguien lo va a necesitar dentro de tres meses? ¿lo que contiene ya está en Git, en el `README` o en una decisión registrada?
- Si el contenido útil de un documento temporal no está en otro lado, pasarlo primero a la documentación permanente y recién después eliminarlo.
- Usar Git para revisar: `git status` para ver lo no versionado y `git clean -n` (simulación) para ver qué se borraría antes de ejecutar nada.
- Agregar al `.gitignore` los patrones de archivos que se repiten (`*.log`, carpetas temporales), para que no vuelvan a aparecer.
- Hacer un commit específico de limpieza, separado de los cambios funcionales, para poder revertirlo si hiciera falta.

> **Regla práctica**
>
> Todo archivo que se crea para ayudar a hacer una tarea y no es parte del resultado se elimina al terminarla. Si resulta útil, se convierte en documentación permanente o se descarta.

## 9. Lista de verificación

| Área | Práctica | Momento |
|---|---|---|
| Versionado | Repositorio en GitHub; secretos y datos reales fuera; commits pequeños. | Siempre |
| Entornos | Una fuente de código; probar en la app de prueba; a producción solo lo probado. | Cada cambio |
| Working copies | Una por funcionalidad; fusionar probado; no acumular. | Cada cambio |
| Importación | Exportar y hacer commit antes de importar; avisar a los usuarios. | Antes de importar |
| IA y base de datos | Conexiones con nombre, mínimo privilegio, aprobación para lo destructivo. | Configuración |
| Reglas de la IA | `CLAUDE.md` con entorno, límites y reglas aprendidas. | Inicio y revisión |
| Skills | Usar el skill oficial de Oracle; la autoridad final es `apex validate`. | Al editar |
| Modelos | Sonnet para el día a día; Opus para diseño, riesgo y depuración difícil; Haiku para lo mecánico. Ajustar el esfuerzo antes de cambiar. | Cada tarea |
| Interfaz | Componentes nativos primero; HTML/CSS propio solo con justificación. | Al diseñar |
| Contexto | Un tema por conversación; compactar al mismo tema, bifurcar para reutilizar contexto y renombrar cada bifurcación. | Durante la sesión |
| Decisiones | Registrar con alternativas y consecuencias; marcar las superadas. | Al decidir |
| Verificación | Comprobar en la aplicación real; probar un caso antes de replicar. | Antes de cerrar |
| Limpieza | Proponer, revisar y eliminar logs, Markdown y exports sobrantes. | Al terminar |
