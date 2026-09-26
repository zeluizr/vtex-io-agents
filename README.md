# vtex-io-agents

**Cinco subagentes de Claude Code para VTEX IO y el servidor MCP `vtex-io-mcp`, en un solo plugin. Los agentes
exploran, revisan, verifican, diseñan integraciones y buscan documentación; el servidor les da las props de los
blocks, las APIs y la documentación de VTEX sin salir del editor.**

Construido sobre las [skills oficiales de VTEX](https://github.com/vtex/skills). Gratis, de código abierto y
continuación de `vtex-agents`, que se retiró junto con su marketplace.

[![licencia](https://badgen.net/github/license/zeluizr/vtex-io-agents?color=142032)](./LICENSE)
[![último commit](https://badgen.net/github/last-commit/zeluizr/vtex-io-agents)](https://github.com/zeluizr/vtex-io-agents/commits)
[![estrellas](https://badgen.net/github/stars/zeluizr/vtex-io-agents)](https://github.com/zeluizr/vtex-io-agents/stargazers)

Versión 0.2.0, según `.claude-plugin/marketplace.json`.

---

## Qué incluye

| Agente                       | Qué hace                                                                                    | Skills que precarga                                                                                                                     | Herramientas del servidor MCP                                             | Modelo    |
| ---------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | --------- |
| `vtex-explorer`              | Mapea la estructura de una app VTEX IO, sus bloques y los contratos que expone. Read-only.  | `vtex-io-app-contract`, `vtex-io-render-runtime-and-blocks`                                                                             | `lookup-block-props`, `explain-concept`                                   | haiku     |
| `vtex-reviewer`              | Revisa el diff antes del publish: seguridad, políticas del manifest, performance, contrato. | `vtex-io-security-boundaries`, `vtex-io-auth-and-policies`, `vtex-io-application-performance`, `architecture-well-architected-commerce` | `lookup-vtex-api`, `search-concepts`, `explain-concept`                   | `inherit` |
| `vtex-qa`                    | Corre build, lint y tests, y devuelve solo las fallas con archivo y stack.                  | `vtex-io-app-contract`, `vtex-io-storefront-theme-versioning`                                                                           | ninguna, a propósito                                                      | sonnet    |
| `vtex-integration-architect` | Diseña y audita integraciones con ERP y marketplaces, con los trade offs explícitos.        | `vtex-io-client-integration`, `vtex-io-events-and-workers`, `marketplace-catalog-sync`, `marketplace-order-hook`                        | `lookup-vtex-api`, `search-concepts`, `explain-concept`, `search-courses` | opus      |
| `vtex-docs-researcher`       | Busca en la documentación de VTEX y cita siempre la fuente.                                 | ninguna, a propósito                                                                                                                    | `search-concepts`, `explain-concept`, `search-courses`, `lookup-vtex-api` | haiku     |

El servidor MCP es [vtex-io-mcp](https://github.com/zeluizr/vtex-io-mcp), ejecutado con `npx -y vtex-io-mcp`. Trae
dentro del paquete 391 documentos de VTEX, los 10 cursos oficiales de VTEX IO, la referencia de las APIs REST y las
props de los blocks de Store Framework; no llama a APIs externas ni pide credenciales. Además de lo que usan los
agentes, deja en la conversación principal las herramientas de scaffolding (`scaffold-vtex-app`,
`scaffold-node-service`, `scaffold-graphql`, `add-block`) y los resources `vtex://concepts` y `vtex://courses`.

Ningún agente precarga más de cuatro skills ni tiene `Write` o `Edit`. El razonamiento detrás de cada decisión está
en [docs/agentes.md](docs/agentes.md).

## Cómo funciona

1. **La `description`** de cada agente es lo que Claude lee para decidir a quién delegar. Están escritas para que la
   delegación ocurra sola: "úsalo proactivamente antes de cualquier cambio estructural", "úsalo antes de
   `vtex publish`".
2. **Las skills precargadas** entran completas al contexto del agente al arrancar. Las demás skills de VTEX siguen
   disponibles bajo demanda con el tool `Skill`.
3. **El servidor MCP** arranca con el plugin. Cada agente declara nombre por nombre las herramientas que usa, con el
   prefijo `mcp__plugin_vtex-io-agents_vtex-io__`; si el servidor no está en la sesión, el agente sigue con lo que
   tiene y lo dice.

## Requisitos

- Claude Code con soporte de plugins.
- Node `>= 18`, para que `npx` ejecute el servidor MCP.
- Las skills oficiales de VTEX instaladas, con `gh skill install vtex/skills` o `npx skills add vtex/skills`. Sin
  ellas los agentes funcionan, pero degradados: Claude Code omite las skills que no encuentra y conserva el flujo de
  cada agente, sin el conocimiento de VTEX que hace precisas sus respuestas. `--scope user` las deja disponibles en
  todos los proyectos.

## Instalación

```bash
/plugin marketplace add zeluizr/vtex-io-agents
/plugin install vtex-io-agents@vtex-io-agents
```

Reinicia Claude Code o corre `/reload-plugins`. Los agentes aparecen en el typeahead de `@` como
`vtex-io-agents:vtex-reviewer`, y el servidor MCP como `vtex-io`.

Si ya tenías `vtex-io-mcp` agregado con `claude mcp add vtex-io`, quítalo con `claude mcp remove vtex-io` para no
tener el servidor dos veces.

## Uso

**Por mención.**

```
@vtex-io-agents:vtex-reviewer revisa lo que acabo de cambiar en node/clients
```

**Por lenguaje natural.** No hace falta nombrarlos.

```
antes de publicar, corre las verificaciones y revisa el diff
```

**Por sesión completa.**

```bash
claude --agent vtex-io-agents:vtex-integration-architect
```

**El servidor MCP, directo.** "¿Qué props acepta `flex-layout.row`?", "arma un servicio Node con una ruta pública
`GET /_v/order/:orderId`", "busca en la documentación cómo funcionan las CSS Handles". Las herramientas y los
resources están en el [README de vtex-io-mcp](https://github.com/zeluizr/vtex-io-mcp#herramientas).

### Fork local de la documentación

`vtex-docs-researcher` busca primero en el servidor MCP y después en un fork local de
[vtexdocs/dev-portal-content](https://github.com/vtexdocs/dev-portal-content), si lo encuentra. Para dárselo:

```bash
git clone https://github.com/vtexdocs/dev-portal-content.git ~/dev-portal-content
claude --add-dir ~/dev-portal-content
```

## Actualizar y desinstalar

| Acción      | Comando                                           |
| ----------- | ------------------------------------------------- |
| Actualizar  | `/plugin marketplace update vtex-io-agents`       |
| Desinstalar | `/plugin uninstall vtex-io-agents@vtex-io-agents` |

## Estructura

```
.claude-plugin/marketplace.json       catálogo del marketplace de Claude Code
plugins/vtex-io-agents/
	.claude-plugin/plugin.json          manifiesto del plugin; mcpServers apunta a .mcp.json
	.mcp.json                           el servidor MCP vtex-io: npx -y vtex-io-mcp
	agents/                             los cinco agentes, uno por archivo
docs/agentes.md                       por qué cada agente es como es
scripts/verificar.sh                  gate de este repositorio
scripts/probar-mcp.mjs                arranca el servidor MCP y comprueba las herramientas que usan los agentes
```

## Desarrollo

```bash
git clone https://github.com/zeluizr/vtex-io-agents.git
cd vtex-io-agents
npm ci
./scripts/verificar.sh
node scripts/probar-mcp.mjs
```

`scripts/verificar.sh` es el gate, el mismo que corre el CI: JSON válido, la misma versión en los tres manifiestos,
la configuración del servidor MCP, el frontmatter de cada agente (nombre igual al archivo, `description`, tool
`Skill` presente, sin `Write` ni `Edit`, máximo cuatro skills precargadas, prefijo MCP propio y skills que existen en
`vtex/skills`), `claude plugin validate --strict`, Prettier, `bash -n` y una búsqueda de rastros de los proyectos de
origen. `scripts/probar-mcp.mjs` arranca el servidor por stdio y falla si una herramienta que un agente declara no
existe. El CI suma la búsqueda de secretos con gitleaks y el control del flujo de ramas.

Para probar el plugin sin publicarlo:

```bash
claude --plugin-dir ./plugins/vtex-io-agents
```

La versión vive en tres lugares: arriba en `.claude-plugin/marketplace.json`, en la entrada del plugin del mismo
archivo y en `plugins/vtex-io-agents/.claude-plugin/plugin.json`. Se sube en los tres a la vez; sin subirla, Claude
Code no entrega el cambio a quien ya tiene el plugin.

## Contribuir

Conventional Commits. Las ramas salen de `dev` y el pull request va contra `dev`, nunca contra `qa` ni contra `main`;
la promoción es merge de la rama entera. Un agente nuevo tiene que pasar la pregunta de
[docs/agentes.md](docs/agentes.md#por-qué-no-hay-más-agentes): ¿genera ruido intermedio que la conversación principal
no necesita ver? Si no, no hace falta.

## Licencia

[MIT](./LICENSE)

_Hecho con amor y café por [zeluizr](https://github.com/zeluizr) y con la ayuda de [Claude](https://claude.ai/referral/Cz_UimA0NQ) ☕_
