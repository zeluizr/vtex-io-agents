# Los cinco agentes y el servidor MCP, y por qué son así

Este documento explica las decisiones de diseño detrás de cada agente del paquete. No es documentación de uso, para eso está el [README](../README.md). Es el razonamiento: por qué cada agente corre en el modelo que corre, por qué carga esas skills y no otras, y por qué casi todos son read-only.

## Los principios que aplican a los cinco

Antes de entrar agente por agente, cinco decisiones que atraviesan todo el paquete.

### Un subagente vale por lo que no devuelve

Un subagente corre en su propia ventana de contexto y devuelve a la conversación principal solo su respuesta final. Todo lo que leyó, todo lo que grepeó, toda la salida de los comandos que ejecutó se queda en su contexto y se descarta.

Esa es la única razón por la que un subagente vale su costo. Si una tarea no genera ruido intermedio, delegarla es más caro que hacerla en la conversación principal, porque pagas el arranque en frío de un agente que no sabe nada de lo que ya conversaron. Los cinco agentes de este paquete existen porque su trabajo produce mucho material intermedio que nadie necesita volver a leer: árboles de directorios, diffs completos, salidas de build, páginas de documentación.

Por eso cada uno tiene una regla explícita de formato de salida, y por eso `vtex-qa` es el caso más extremo del paquete: corre comandos que escupen cientos de líneas y devuelve una sola cuando todo pasa.

### Máximo cuatro skills precargadas

El campo `skills` del frontmatter inyecta el contenido completo de cada skill en el contexto del subagente al arrancar, no solo su descripción. Precargar diez skills significa que el agente empieza a trabajar con la ventana ya cargada de conocimiento que quizá no aplique al caso.

El límite de cuatro es arbitrario en el número pero no en la intención: precargas lo que el agente va a necesitar casi siempre, y dejas el resto para que lo pida si lo necesita. Los cinco agentes tienen el tool `Skill` habilitado, así que las 45 skills oficiales de VTEX siguen disponibles bajo demanda.

La consecuencia práctica es que la elección de las skills precargadas es una apuesta sobre el caso típico de cada agente, no sobre su caso máximo.

### Read-only por defecto

Ninguno de los cinco tiene `Write` ni `Edit`. Dos de ellos tienen `Bash`, y sus instrucciones acotan explícitamente para qué.

No es paranoia, es división del trabajo. Un agente que puede escribir se convierte en un desarrollador paralelo que no ve la conversación completa, y sus cambios llegan a la conversación principal como un hecho consumado que hay que revisar. Un agente que solo lee devuelve información, y la decisión de qué hacer con esa información se queda donde está el contexto acumulado.

### El servidor MCP es fuente, no cerebro

El plugin trae el servidor MCP `vtex-io`, que es [vtex-io-mcp](https://github.com/zeluizr/vtex-io-mcp) ejecutado con `npx -y vtex-io-mcp`. Viaja con su conocimiento dentro del paquete: 391 documentos de VTEX, los 10 cursos oficiales de VTEX IO, la referencia de las APIs REST y las props de los blocks de Store Framework. No llama a ninguna API externa ni pide credenciales.

La decisión es qué agente recibe qué herramienta, y la regla fue la misma que para las skills: cada agente recibe solo las herramientas que su caso típico necesita, y ninguna recibe las de scaffolding. Las tres herramientas que generan código (`scaffold-vtex-app`, `scaffold-node-service`, `scaffold-graphql`) y `add-block` quedan para la conversación principal, que es donde se escribe código. Un agente read-only con herramientas de scaffolding sería una contradicción.

Las herramientas MCP de un plugin llevan el prefijo `mcp__plugin_vtex-io-agents_vtex-io__`, y así aparecen en el campo `tools` de cada agente. Ese campo es una lista cerrada: lo que no está ahí, el agente no lo ve. Por eso cada agente declara nombre por nombre las herramientas que usa, y por eso el gate del repositorio arranca el servidor de verdad y comprueba que cada nombre exista. Si `vtex-io-mcp` renombra una herramienta, falla el CI de este repositorio antes de fallar en la sesión de alguien.

Todos los agentes que usan el servidor tienen la misma instrucción de degradación: si las herramientas no aparecen en la sesión, siguen con lo que tienen y lo dicen en una línea. El servidor mejora la respuesta; su ausencia no la impide.

### La description es el router, no la documentación

Claude decide a qué agente delegar leyendo el campo `description` de cada uno. No lee el cuerpo del archivo hasta que ya decidió delegar.

Por eso las cinco descripciones dicen cuándo delegar, no qué hace el agente por dentro. Tres de ellas incluyen una instrucción de proactividad explícita, porque la diferencia entre un agente que se usa y uno que no se usa suele ser esa frase.

## `vtex-explorer`

**Qué problema resuelve.** Nadie debería tocar la estructura de una app VTEX IO sin saber qué builders declara, qué bloques define, qué extiende de un tema base y qué contratos expone hacia afuera. Averiguarlo a mano son veinte llamadas de Glob, Grep y Read que llenan la conversación de árboles de directorios y fragmentos de JSON.

**Por qué haiku.** Es el agente más barato del paquete y el que más se invoca. Su trabajo es de reconocimiento de patrones sobre archivos con formato conocido, no de razonamiento. `manifest.json` declara builders, `store/blocks.json` define bloques, `service.json` declara rutas: identificar eso no necesita un modelo caro. Correrlo en haiku significa que puedes invocarlo sin pensarlo dos veces, que es exactamente lo que la palabra "proactivamente" en su descripción pide.

**Por qué esas dos skills.** `vtex-io-app-contract` le da el vocabulario de la superficie declarativa de una app: builders, dependencies, peerDependencies, policies, identidad. `vtex-io-render-runtime-and-blocks` le da el modelo mental de composición del storefront, que es la mitad del trabajo cuando el proyecto es un tema o una app de componentes.

Vale la pena señalar que aquí hubo un cambio respecto al diseño original. La skill que se había pensado para este agente, `vtex-io-app-structure`, no existe en el repositorio oficial de VTEX. Se verificó nombre por nombre contra `vtex/skills` antes de escribir el archivo, y se eligió `vtex-io-render-runtime-and-blocks` como reemplazo porque es la que cubre literalmente los bloques que la descripción del agente promete mapear.

**Por qué read-only estricto.** Es un cartógrafo. Devuelve un mapa, no un plan de obra. Si además pudiera cambiar cosas, la conversación principal perdería la oportunidad de decidir el cambio con el mapa en la mano.

**Qué recibe del servidor MCP.** `lookup-block-props` y `explain-concept`, nada más. La primera para decir qué es y qué props acepta un block nativo que apareció en `store/blocks.json`; la segunda para nombrar con precisión un builder o una policy. El prompt le prohíbe usarlas para explorar la documentación, que es trabajo del investigador: un cartógrafo barato que se pone a leer documentación deja de ser barato.

**Qué devuelve.** Un resumen estructurado de unas cuarenta líneas: tipo de app, dónde está qué, bloques definidos y extendidos, contratos expuestos, y puntos de atención. Nunca contenido de archivos. Si necesitas el contenido, te da la ruta y las líneas.

## `vtex-reviewer`

**Qué problema resuelve.** El momento más caro para descubrir un problema en VTEX IO es después del publish, cuando la versión ya está publicada y los temas que dependen de tu app ya la resolvieron. Este agente mete una revisión entre escribir el código y publicarlo.

**Por qué `inherit`.** Es el único agente del paquete que hereda el modelo de la conversación principal, y es deliberado. Revisar código es la tarea donde la calidad del modelo se nota más directamente en la calidad del resultado, y también es donde el usuario ya expresó su preferencia al elegir con qué modelo trabajar. Si eligió un modelo caro para escribir el código, quiere ese mismo modelo revisándolo. Si eligió uno barato, forzarle uno caro en la revisión le cambia la economía de la sesión sin avisarle.

**Por qué esas cuatro skills.** Son los cuatro ejes donde un error de VTEX IO cuesta caro en producción. `vtex-io-security-boundaries` y `vtex-io-auth-and-policies` cubren juntas la superficie de seguridad, que en VTEX IO se juega sobre todo en qué token usa cada llamada y qué declara el manifest. `vtex-io-application-performance` cubre el patrón que más se repite en apps de servicio: llamadas en serie que podían ir en paralelo y ausencia de cache donde correspondía. `architecture-well-architected-commerce` es la que aporta la mirada transversal, la que pregunta si el cambio tiene sentido dentro de la arquitectura de comercio y no solo dentro del archivo.

Estas cuatro llenan el cupo del paquete. Es el agente con más contexto precargado, y se justifica porque la revisión es la tarea donde el conocimiento previo evita más falsos negativos.

**Por qué tiene `Bash`.** Porque la primera instrucción del agente es correr `git diff`. Sin eso, revisaría el repositorio entero en lugar de lo que cambió, que es más caro y menos útil. El `Bash` está acotado por instrucción a inspección: git, ls, lectura de fragmentos. Nada que modifique el repositorio, nada de `vtex publish`.

**Qué recibe del servidor MCP.** `lookup-vtex-api`, `search-concepts` y `explain-concept`, con una instrucción precisa: verificar antes de reportar, nunca revisar en su lugar. Un revisor que marca como incorrecta una llamada a la API de pedidos porque recuerda mal el endpoint hace daño; con `lookup-vtex-api` a mano, confirma antes de escribir el hallazgo. Es la razón por la que el prompt dice que un hallazgo verificado vale más que dos de memoria.

**Por qué `memory: project`.** Es el agente que más se beneficia de recordar. Cada proyecto tiene sus convenciones, sus falsos positivos ya discutidos y sus decisiones arquitectónicas aceptadas. Un revisor que vuelve a levantar el mismo hallazgo que el equipo ya descartó deja de ser útil rápido. La memoria de proyecto le permite acumular ese contexto entre sesiones.

La instrucción es explícita en las dos direcciones: consultar la memoria antes de revisar, y actualizarla al terminar con los patrones recurrentes detectados. Y con un límite igual de explícito sobre qué nunca guardar: credenciales, nombres de clientes, datos de tiendas.

**Qué devuelve.** Tres niveles fijos, siempre en el mismo orden y siempre con los mismos nombres: Crítico (bloquea el publish), Aviso (corregir antes del release), Sugerencia. Cada hallazgo con archivo, línea, qué está mal en una frase, y la corrección propuesta. Si un nivel queda vacío, aparece igual con la palabra Ninguno, para que se note la diferencia entre "revisé y no encontré nada crítico" y "no llegué a revisar eso".

## `vtex-qa`

**Qué problema resuelve.** Correr build, lint y tests genera cientos de líneas de salida de las que importan cinco. Ese ruido, en la conversación principal, desplaza contexto que sí importa.

**Por qué sonnet.** Es el punto medio del paquete. Haiku se queda corto porque filtrar una salida de build exige distinguir el error real de las diez líneas de stack que lo rodean, del warning de dependencia que no importa y del test que falló como consecuencia de otro. Opus sería desperdicio, porque una vez identificado el error no hay que razonar sobre él, hay que reportarlo con su archivo y su línea.

**Por qué esas dos skills.** `vtex-io-app-contract` porque la verificación no termina en los scripts de npm: hay que mirar si la versión del manifest acompaña el tipo de cambio. `vtex-io-storefront-theme-versioning` porque el caso que más caro sale es el bump de major.

Ese caso merece explicación. Cuando una app que expone bloques o interfaces sube de major, rompe el contrato con los temas que la consumen, y esos temas se quedan clavados en la versión anterior hasta que alguien los actualice a mano. Es un problema que no aparece en ningún test, no lo detecta el build y no lo ve el lint. Solo lo ve alguien que sabe qué significa un major en VTEX IO, y por eso esa skill está precargada.

**Por qué tiene `Bash` como primer tool.** Es su razón de existir. Los otros tools están para lo que viene después de correr: leer el `package.json` para saber qué scripts existen de verdad, detectar el gestor de paquetes por el lockfile, ubicar el archivo donde reventó.

**Por qué no tiene herramientas MCP.** Es el único agente del paquete sin acceso al servidor, y su prompt lo explica. Su trabajo es correr comandos y filtrar salida; consultar documentación durante una verificación es exactamente el ruido que existe para evitar. Si una falla necesita contexto de VTEX para entenderse, la reporta con archivo y línea y deja la investigación a quien lo invocó.

**Qué devuelve, y esto es lo importante.** Si todo pasa, una línea. Si algo falla, solo las fallas, con archivo, línea, el mensaje recortado a lo esencial y las líneas de stack que apuntan a código del proyecto, descartando las de `node_modules`. Nunca la salida completa de un comando, nunca barras de progreso, nunca la lista de tests que pasaron. Si hay muchas fallas del mismo tipo, las agrupa y dice cuántas son.

Es el agente donde la regla de "vale por lo que no devuelve" se aplica de forma más literal.

## `vtex-integration-architect`

**Qué problema resuelve.** Es el único agente del paquete que emite juicio. Los otros cuatro reportan hechos: esto es lo que hay, esto está mal, esto falló, esto dice la documentación. Este propone una arquitectura y se hace responsable de los trade offs de esa propuesta.

**Por qué opus.** Porque diseñar una integración de catálogo o de pedidos entre VTEX y un ERP es la tarea del paquete donde equivocarse cuesta más y donde el error tarda más en aparecer. Una sincronización mal diseñada no falla el primer día, falla el día que el ERP se cae media hora y nadie sabe qué pedidos se perdieron. Ese tipo de razonamiento, sobre modos de falla que todavía no ocurrieron, es exactamente donde el modelo más capaz se paga solo.

Es también el agente que menos se invoca, así que su costo por sesión es bajo aunque su costo por invocación sea el más alto de los cinco.

**Por qué esas cuatro skills.** Dos del lado de VTEX IO y dos del lado del marketplace. `vtex-io-client-integration` cubre cómo un backend de VTEX IO habla con servicios externos sin reinventar el cliente HTTP. `vtex-io-events-and-workers` cubre el procesamiento asincrónico, que es donde termina viviendo cualquier integración seria una vez que sale del camino feliz. `marketplace-catalog-sync` y `marketplace-order-hook` cubren los dos flujos concretos que la descripción del agente promete: catálogo y pedidos.

Las que quedaron fuera son deliberadas y el agente sabe pedirlas: `marketplace-rate-limiting` cuando el volumen lo amerita, `marketplace-fulfillment` cuando el caso llega hasta la facturación, `masterdata-storage-strategy` cuando aparece la tentación de guardar datos operativos en Master Data.

**Qué recibe del servidor MCP.** Las cuatro herramientas de documentación: `lookup-vtex-api`, `search-concepts`, `explain-concept` y `search-courses`. Es el agente que más las aprovecha, porque una integración se diseña sobre los endpoints reales de pedidos, catálogo, Master Data y logística, y sobre cómo funcionan de verdad los eventos y los workers en VTEX IO. La instrucción es consultar el servidor antes de WebFetch: no depende de la red y responde igual en cada sesión.

**Por qué read-only con `WebFetch`.** No implementa, diseña. Su entregable es una decisión argumentada, no un pull request. El `WebFetch` está para la documentación pública de VTEX y del sistema externo, nunca contra ambientes de una tienda.

**Por qué `memory: project`.** Las decisiones arquitectónicas se toman una vez y se viven durante años. Un agente que no recuerda por qué se eligió pull en lugar de push va a proponer push en la siguiente sesión, y el equipo va a tener que volver a explicar la misma restricción.

**Las dos reglas innegociables.** Nunca proponer una integración sin estrategia de reintento y sin estrategia de reconciliación, que son cosas distintas: el reintento resuelve la falla transitoria del momento, la reconciliación resuelve la deriva acumulada que el reintento no vio. Y siempre explicitar los trade offs en lugar de esconderlos: consistencia contra latencia, push contra pull, idempotencia, acoplamiento, límites de tasa.

**Qué devuelve.** Contexto entendido, opciones consideradas con sus trade offs (mínimo dos, porque una sola opción no es una decisión), recomendación con su porqué, puntos de falla, estrategia de reintento y reconciliación, y qué queda abierto. Esa última sección es la que más valor tiene: las preguntas que el negocio tiene que responder antes de que alguien escriba código.

## `vtex-docs-researcher`

**Qué problema resuelve.** Buscar en la documentación de VTEX consume mucho contexto para devolver poca información. Abres cuatro páginas, la respuesta está en un párrafo de una de ellas, y las otras tres quedan ocupando la conversación para siempre.

**Por qué haiku.** Su trabajo es buscar y citar, no razonar. Encontrar el párrafo correcto y transcribir de dónde salió no necesita un modelo caro. Y como es el agente que más veces se invoca por sesión en un equipo que está aprendiendo VTEX, que sea barato importa.

**Por qué no precarga ninguna skill, y esto es lo más contraintuitivo del paquete.** Es deliberado. Un agente de investigación que arranca con cuatro skills en contexto arranca con una hipótesis sobre dónde está la respuesta, y esa hipótesis sesga la búsqueda. Arrancar vacío le deja la ventana entera para la documentación que la pregunta efectivamente necesita.

Sigue teniendo el tool `Skill`, así que si la pregunta cae claramente dentro de una skill, la invoca. La diferencia es que la carga porque la pregunta lo pidió, no porque estaba ahí desde antes.

**El orden de búsqueda.** Primero el servidor MCP `vtex-io`, que viene con el plugin: 391 documentos, 10 cursos y la referencia de las APIs REST, sin red y con resultados iguales en cada sesión. `search-concepts` encuentra el ID, `explain-concept` lee el documento entero; `search-courses` y `lookup-vtex-api` cubren cursos y APIs. Segundo, un fork local de `vtexdocs/dev-portal-content` si está disponible, porque buscar con Grep sobre texto plano es exhaustivo y reproducible. Tercero, la documentación que ya viva dentro del propio repositorio del proyecto. Y solo después, `WebFetch` contra la documentación pública.

El servidor MCP va primero y no el fork por una razón práctica: el fork hay que clonarlo y apuntarlo con `--add-dir`, y casi nadie lo tiene; el servidor lo tiene todo el que instaló el plugin.

Ese orden es la razón por la que el README dedica una sección a explicar cómo apuntar el fork local con `--add-dir`. El agente funciona sin él, pero funciona bastante mejor con él.

**Por qué `memory: user` y no `project`.** Es el único del paquete con memoria de usuario. El mapa de la documentación de VTEX no cambia de un cliente a otro: dónde vive cada tema dentro de `dev-portal-content`, qué búsquedas dieron resultado, qué URLs son estables. Eso es conocimiento que vale la pena acumular una sola vez y arrastrar a todos tus proyectos.

**La regla de cita.** Toda afirmación va con su fuente: herramienta e ID cuando viene del servidor MCP, ruta y línea cuando viene de un archivo local, URL completa cuando viene de la web. Si no encontró la respuesta, lo dice. Si aporta algo de su propio conocimiento, lo marca aparte y de forma visible como no verificado en la documentación.

Esa última parte es la que convierte al agente en algo en lo que se puede confiar. Un investigador que rellena los huecos con conocimiento previo presentado como si viniera de la documentación es peor que no tener investigador.

## Por qué no hay más agentes

La pregunta que decide si vale la pena un agente nuevo es siempre la misma: ¿esta tarea genera ruido intermedio que la conversación principal no necesita ver?

Si la respuesta es no, el agente sale más caro que el trabajo. Un subagente arranca en frío, sin el contexto de lo que ya se habló, lo que ya se probó y lo que ya se descartó. Pagas ese arranque a cambio de que el ruido se quede afuera. Cuando no hay ruido que dejar afuera, solo pagas.

Por eso no hay un agente "Developer" ni un agente "Designer". Escribir código es la tarea que más se beneficia del contexto acumulado y menos ruido intermedio genera. Pertenece a la conversación principal.

Y por eso tampoco un agente es siempre la respuesta correcta. Si lo que necesitas es una restricción sobre un directorio concreto, una regla en `.claude/rules/` con `paths:` acotado es más barata y más precisa, porque se activa sola cuando corresponde y no gasta un contexto entero. Si lo que necesitas es que el equipo comparta convenciones, eso va en `CLAUDE.md`.

Los agentes son para el trabajo ruidoso, acotado y repetitivo. Cinco alcanzan.
