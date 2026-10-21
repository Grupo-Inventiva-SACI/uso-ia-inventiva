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
- [6. Documentación y rigor técnico](#6-documentación-y-rigor-técnico)
- [7. Cierre de tarea y limpieza de archivos sobrantes](#7-cierre-de-tarea-y-limpieza-de-archivos-sobrantes)
- [8. Lista de verificación](#8-lista-de-verificación)

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

### 3.3 Elegir el modelo según la tarea

- Modelos más capaces para diseño, arquitectura, depuración difícil y revisión de código.
- Modelos más rápidos y económicos para tareas mecánicas: generar páginas repetitivas, renombrar, buscar, formatear.
- Si una tarea se atasca con un modelo, probar con otro antes de insistir con más intentos.

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

La calidad de las respuestas depende de lo que la IA tiene presente. Una conversación demasiado larga o mezclada con varios temas degrada los resultados.

### 5.1 Memoria permanente en el repositorio

- `CLAUDE.md`: lo que toda sesión nueva debe saber (entornos, convenciones, reglas, qué no tocar). Mantenerlo corto y actualizado.
- Documentación de decisiones: lo decidido queda escrito con sus alternativas, así una conversación nueva retoma sin repetir el razonamiento.
- Bitácora de hitos y `RUNBOOK`: qué se hizo, en qué estado quedó y cómo se opera.

### 5.2 Compactar y bifurcar conversaciones

- **Compactar** cuando la conversación crece: se conserva lo importante y se libera espacio para seguir trabajando.
- **Bifurcar** para explorar una alternativa sin contaminar la línea principal, o para tareas independientes entre sí (por ejemplo, una página y un script de base de datos).
- **Empezar una conversación nueva** al cambiar de tarea. Si lo importante quedó en el `CLAUDE.md` y en la documentación, no se pierde nada.
- Pedir a la IA que resuma el estado y las decisiones pendientes antes de cerrar una sesión larga.

### 5.3 Delegar la exploración

- Las búsquedas amplias (por ejemplo, localizar todas las páginas que usan cierto componente) se delegan a subagentes, para no llenar la conversación principal con resultados intermedios.

## 6. Documentación y rigor técnico

### 6.1 Registrar decisiones

- Documentar una decisión solo cuando hubo más de un camino posible y la elección condiciona el trabajo futuro.
- Cada registro lleva: contexto, alternativas evaluadas, decisión, consecuencias y estado (propuesta, aceptada, superada).
- Cuando una decisión resulta ser incorrecta, no se borra: se marca como superada y se enlaza a la que la reemplaza.
- Incluir la consulta SQL que permite verificar la decisión.

### 6.2 Verificar en el producto real

- Una validación correcta no garantiza que la pantalla se vea bien: abrir la aplicación y comprobarlo.
- Los fallos silenciosos son los más caros (un slot de región inválido, un item sin plantilla de label, un branch que hace rollback). Documentarlos con su síntoma y su causa.
- Probar un caso en una página antes de replicar el patrón en veinte.

### 6.3 Modelado de datos

- Garantizar las reglas de negocio en la base: índices únicos parciales, CHECK, claves foráneas. La aplicación no es la última línea de defensa.
- Reemplazar el texto libre por catálogos cuando el dato se usa para filtrar o agrupar.
- Definir la identidad de una entidad una sola vez (por ejemplo, en una columna virtual) y reutilizarla en vistas y paquetes.
- Las páginas consultan **vistas**, nunca tablas base, y la lógica vive en paquetes PL/SQL.
- Ante un posible falso positivo, preferir el falso negativo: un aviso equivocado cuesta la confianza en todos los demás.
- Cuidado con `ON DELETE CASCADE`: puede eliminar datos relacionados en silencio. Si la regla de negocio lo prohíbe, controlarlo explícitamente.

### 6.4 Seguridad

- Las contraseñas nunca se guardan en claro, ni se escriben en logs ni pasan por el estado de sesión: salt por usuario y hash iterado.
- La autorización se valida en la página y en el proceso, no solo en el botón.
- Los authorization schemes de rol se cachean una vez por vista de página (Once per page view), no por sesión; así quitar un rol tiene efecto inmediato.
- Activar Session State Protection en toda página que reciba un identificador por URL.
- Registrar en una tabla de auditoría de solo inserción toda acción sobre cuentas ajenas.

## 7. Cierre de tarea y limpieza de archivos sobrantes

Trabajar con IA genera muchos archivos intermedios: registros de ejecución, exportaciones de prueba, planes temporales, informes de revisión y borradores. Al terminar una tarea, esos restos no aportan nada y ensucian el repositorio, el contexto de la IA y las búsquedas.

### 7.1 Qué suele sobrar

- Archivos de registro (`*.log`) y salidas de consola guardadas.
- Documentos Markdown de trabajo que ya cumplieron su función: planes temporales, informes de sesión, críticas o revisiones puntuales, notas de depuración.
- Exportaciones de prueba y copias de respaldo antiguas de aplicaciones, cuando ya existe el commit equivalente en Git.
- Renders, capturas e imágenes intermedias; carpetas temporales (`tmp`, `__pycache__`).
- Scripts de un solo uso que ya se ejecutaron y no se volverán a necesitar.
- Duplicados: versiones `v1`, `v2`, `(1)` de un mismo archivo.

### 7.2 Qué se conserva

- `README`, `CLAUDE.md`, la documentación de decisiones, la bitácora de hitos y el `RUNBOOK`.
- Migraciones y scripts reutilizables de base de datos.
- Las herramientas de verificación y los generadores que se siguen usando.
- Los respaldos que son la única copia de algo (por ejemplo, un dato previo a una migración).

### 7.3 Cómo hacerlo

- Al cerrar la tarea, **pedir a la IA que proponga una lista de candidatos a eliminar**, con el motivo de cada uno. No debe borrar por su cuenta.
- Una persona revisa la lista y confirma. Para cada archivo, preguntar: ¿alguien lo va a necesitar dentro de tres meses? ¿lo que contiene ya está en Git, en el `README` o en una decisión registrada?
- Si el contenido útil de un documento temporal no está en otro lado, pasarlo primero a la documentación permanente y recién después eliminarlo.
- Usar Git para revisar: `git status` para ver lo no versionado y `git clean -n` (simulación) para ver qué se borraría antes de ejecutar nada.
- Agregar al `.gitignore` los patrones de archivos que se repiten (`*.log`, carpetas temporales), para que no vuelvan a aparecer.
- Hacer un commit específico de limpieza, separado de los cambios funcionales, para poder revertirlo si hiciera falta.

> **Regla práctica**
>
> Todo archivo que se crea para ayudar a hacer una tarea y no es parte del resultado se elimina al terminarla. Si resulta útil, se convierte en documentación permanente o se descarta.

## 8. Lista de verificación

| Área | Práctica | Momento |
|---|---|---|
| Versionado | Repositorio en GitHub; secretos y datos reales fuera; commits pequeños. | Siempre |
| Entornos | Una fuente de código; probar en la app de prueba; a producción solo lo probado. | Cada cambio |
| Working copies | Una por funcionalidad; fusionar probado; no acumular. | Cada cambio |
| Importación | Exportar y hacer commit antes de importar; avisar a los usuarios. | Antes de importar |
| IA y base de datos | Conexiones con nombre, mínimo privilegio, aprobación para lo destructivo. | Configuración |
| Reglas de la IA | `CLAUDE.md` con entorno, límites y reglas aprendidas. | Inicio y revisión |
| Skills | Usar el skill oficial de Oracle; la autoridad final es `apex validate`. | Al editar |
| Modelos | Más capaz para diseñar y depurar; más ágil para tareas mecánicas. | Cada tarea |
| Interfaz | Componentes nativos primero; HTML/CSS propio solo con justificación. | Al diseñar |
| Contexto | Compactar, bifurcar o abrir una conversación nueva según el caso. | Durante la sesión |
| Decisiones | Registrar con alternativas y consecuencias; marcar las superadas. | Al decidir |
| Verificación | Comprobar en la aplicación real; probar un caso antes de replicar. | Antes de cerrar |
| Limpieza | Proponer, revisar y eliminar logs, Markdown y exports sobrantes. | Al terminar |
