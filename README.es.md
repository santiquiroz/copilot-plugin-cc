# copilot-plugin-cc

Delega tareas mecánicas de codificación desde [Claude Code](https://claude.com/claude-code)
a [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli)
en modo autónomo no interactivo.

Claude Code se mantiene como el orquestador — escribe la lógica de dominio, define cada
contrato de subtarea y revisa los diffs. Copilot CLI absorbe el trabajo puramente
mecánico (código estándar, renombramientos, limpieza de código muerto, especificaciones simples, mapeo
de DTOs, descripciones de PR) en segundo plano, para que tu contexto y tokens de Claude
se dediquen únicamente al trabajo que solo Claude puede hacer.

Inspirado en la estructura de [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc).
**No afiliado a GitHub, Microsoft, OpenAI ni Anthropic.**

> Read this in English: [README.md](README.md)

## Requisitos

- Claude Code
- Una suscripción a GitHub Copilot
- GitHub Copilot CLI ≥ **1.0.67** (`npm install -g @github/copilot`)

## Instalación

En Claude Code:

```
/plugin marketplace add santiquiroz/copilot-plugin-cc
/plugin install copilot@copilot-plugin-cc
```

Luego verifica tu entorno:

```
/copilot:setup
```

## Uso

Delegación explícita:

```
/copilot:rescue remove all unused imports under src/ and fix the import order
/copilot:rescue --background generate boilerplate test specs for src/services/user-mapper.ts
/copilot:rescue --model claude-sonnet-5 rename WidgetFactory to WidgetBuilder across the repo
```

Delegación proactiva: el agente `copilot-rescue` se describe a sí mismo para que Claude
Code lo seleccione automáticamente para tareas mecánicas. Para integrarlo en tus propias
reglas de delegación, copia el bloque de
[docs/claude-md-snippet.md](docs/claude-md-snippet.md) en tu `CLAUDE.md`.

Patrones de orquestación completa — la división Codex/Copilot/inline, el patrón paralelo,
límites de WIP y la cadena de fallback de cuota — están en
[docs/delegation-guide.md](docs/delegation-guide.md).

## Modelo de seguridad

Cada tarea reenviada ejecuta Copilot CLI con un conjunto de flags acotado:

```
copilot -p "<task>" -s \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' --deny-tool='shell(git reset)'
```

Las reglas de negación prevalecen sobre las reglas de permitir — incluso bajo `--allow-all` — así que una
tarea mecánica puede escribir archivos y usar git local, pero nunca puede eliminar archivos, hacer push o
restablecer estado compartido. `--allow-all` solo se usa cuando una tarea genuinamente necesita una
herramienta fuera de git/write y el usuario lo confirmó.

## Problemas conocidos del CLI que este plugin mitiga

| Problema | Solución integrada |
|---|---|
| Bucle infinito de autopiloto en tareas bloqueadas externamente ([copilot-cli#2969](https://github.com/github/copilot-cli/issues/2969)) | Se evita `--autopilot` para tareas acotadas; cuando se usa, `--max-autopilot-continues <N>` siempre se fija explícitamente |
| Reanudar después de un límite de velocidad puede colgar | En salida "rate limit" + ~10s de silencio: termina el proceso e informa, nunca esperes; relanza fresco sin `--continue` |

## Qué hay en el plugin

| Componente | Propósito |
|---|---|
| `agents/copilot-rescue.md` | Agente forwarder delgado — una llamada `copilot -p`, salida devuelta textualmente |
| `/copilot:rescue` | Delega una tarea explícitamente (`--background`, `--wait`, `--model <name>`) |
| `/copilot:setup` | Verifica la instalación de CLI, versión mínima, autenticación y opciones de fijación de modelo |
| `docs/delegation-guide.md` | Guía completa de orquestación multi-agente |
| `docs/claude-md-snippet.md` | Bloque listo para copiar en CLAUDE.md |
| `.codex-plugin/plugin.json` | Manifiesto de plugin para Codex CLI (instalación nativa experimental) |
| `skills/copilot-rescue/SKILL.md` | Skill para Codex — misma lógica de reenvío, Codex ejecuta la llamada de shell directamente (sin capa de subagente) |
| `docs/agents-md-snippet.md` | Bloque listo para copiar en `AGENTS.md` para usuarios de Codex |

## Codex CLI

La misma idea, para [Codex CLI](https://developers.openai.com/codex/): delegar
trabajo mecánico a GitHub Copilot CLI, para que el razonamiento de Codex se
dedique solo al trabajo que únicamente él puede hacer. Se distribuye como una
**skill** de Codex (`skills/copilot-rescue/SKILL.md`) en vez de un subagente —
Codex ejecuta el comando `copilot -p ...` reenviado él mismo, ya que Codex no
tiene una capa separada de subagente/Task.

### Requisitos

- Codex CLI
- Una suscripción a GitHub Copilot
- GitHub Copilot CLI ≥ **1.0.67** (`npm install -g @github/copilot`)

### Instalación

**Manual (funcionamiento confirmado):**

```bash
mkdir -p ~/.agents/skills
cp -r skills/copilot-rescue ~/.agents/skills/copilot-rescue
```

O solo a nivel de repo: copia en `<tu-repo>/.agents/skills/copilot-rescue/` en su lugar.

**Marketplace de plugins nativo (experimental — el sistema de plugins de Codex
es nuevo; abre un issue si las rutas no coinciden con tu versión de Codex CLI):**

```
codex plugin marketplace add santiquiroz/copilot-plugin-cc
```

Luego abre el navegador de plugins (`/plugins` dentro de Codex) e instala `copilot`.

### Uso

Codex empareja skills de forma implícita por su `description`, o puedes invocarla
explícitamente. Pide una tarea mecánica y Codex debería elegir `copilot-rescue`
por su cuenta; para integrar delegación proactiva en tus propias instrucciones,
copia el bloque de [docs/agents-md-snippet.md](docs/agents-md-snippet.md) en tu
`AGENTS.md`.

### Modelo de seguridad

Igual que en el lado de Claude Code — ver [Modelo de seguridad](#modelo-de-seguridad)
arriba. La skill ejecuta Copilot CLI con el mismo conjunto acotado de flags allow/deny.

## Licencia

[MIT](LICENSE)
