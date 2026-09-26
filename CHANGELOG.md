# Changelog

Todos los cambios de vtex-io-agents. El formato sigue [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y
las versiones siguen [SemVer](https://semver.org/lang/es/).

El paquete continúa a `vtex-agents` (versiones 0.1.0 y 0.1.1, del marketplace `inmmerce`, ya retirado). Los cinco
agentes son los mismos; cambian el nombre, el repositorio y la incorporación del servidor MCP.

## [Sin publicar]

### Agregado

- El plugin trae el servidor MCP `vtex-io` ([vtex-io-mcp](https://github.com/zeluizr/vtex-io-mcp)), ejecutado con
  `npx -y vtex-io-mcp`: scaffolding de apps, servicios Node y GraphQL, props de blocks de Store Framework y 391
  documentos, 10 cursos y la referencia de las APIs REST de VTEX dentro del paquete.
- `vtex-docs-researcher` busca primero en el servidor MCP (`search-concepts`, `explain-concept`, `search-courses`,
  `lookup-vtex-api`) y después en el fork local, el repositorio y la web. Cita la herramienta y el ID como fuente.
- `vtex-reviewer` verifica endpoints y conceptos con `lookup-vtex-api`, `search-concepts` y `explain-concept` antes de
  reportar un hallazgo.
- `vtex-integration-architect` consulta las APIs y los conceptos de VTEX IO en el servidor MCP antes de recurrir a
  WebFetch.
- `vtex-explorer` usa `lookup-block-props` y `explain-concept` para describir contratos con precisión.
- `vtex-qa` queda sin herramientas MCP a propósito, y su prompt lo explica.
- Gate del repositorio (`scripts/verificar.sh`): JSON válido, misma versión en los tres manifiestos, configuración del
  servidor MCP, frontmatter de los agentes (nombre, description, `Skill` presente, sin `Write` ni `Edit`, máximo cuatro
  skills precargadas, prefijo MCP propio, skills existentes en `vtex/skills`), `claude plugin validate --strict`,
  Prettier, `bash -n` y búsqueda de rastros de los proyectos de origen.
- `scripts/probar-mcp.mjs`: arranca el servidor MCP por stdio y comprueba que cada herramienta que usa un agente exista.
- CI en GitHub Actions con el gate, la prueba del servidor MCP, gitleaks y el control del flujo `dev → qa → main`.

### Cambiado

- Nombre del plugin y del marketplace: `vtex-io-agents`, en `zeluizr/vtex-io-agents`, junto a `vtex-io-mcp` y
  `vtex-io-snippets`.
- README y `docs/agentes.md` reescritos para el paquete con servidor MCP.
- Manifiestos, JSON y Markdown con tabulación de ancho 2 y Prettier.
