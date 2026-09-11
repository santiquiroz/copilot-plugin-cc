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
- GitHub Copilot CLI ≥ **1.0.83** recomendado (`npm install -g @github/copilot`); 1.0.67 sigue siendo el mínimo para `--model`

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
/copilot:rescue --credits 50 generate boilerplate test specs for src/services/user-mapper.ts
/copilot:rescue --allow-shell dotnet add null-guard tests to OrderMapperTests.cs and make sure dotnet test passes
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
copilot -p "$(cat <<'COPILOT_TASK'
<task>
COPILOT_TASK
)" -s --no-ask-user --max-ai-credits 30 \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' \
  --deny-tool='shell(git reset)' --deny-tool='shell(git clean)' \
  --deny-tool='shell(git checkout)' --deny-tool='shell(git restore)' \
  --deny-tool='shell(git switch)' --deny-tool='shell(git rm)' \
  --deny-tool='shell(git stash)' --deny-tool='shell(git worktree)' \
  --deny-tool='shell(git submodule)' --deny-tool='shell(git config)'
```

La tarea pasa por un heredoc con comillas, nunca en línea entre comillas
dobles, así que los backticks, `$` y comillas llegan literales a Copilot en vez
de que el shell los ejecute o los altere. El límite predeterminado es de 30
créditos de IA (el mínimo del CLI), `--no-ask-user` evita que el agente se
detenga esperando a una persona y las reglas de negación prevalecen: mientras
solo git esté permitido, una tarea mecánica puede escribir archivos y usar git
local, pero no puede ejecutar directamente `rm` ni los subcomandos destructivos
de git (`push`, `reset`, `clean`, `checkout`, `restore`, `switch`, `rm`,
`stash`, `worktree`, `submodule`, `config`). La coincidencia es por subcomando
de primer nivel, y git mismo puede ejecutar código con alias
(`git -c alias.x='!cmd' x`) o con hooks que Copilot escriba, así que es una
barrera, no un sandbox: revisa `git status`, `git log` y `git reflog` después de
cada corrida. `--credits <N>` cambia el límite.

Por defecto solo puede ejecutarse `git`; en modo `-p` cualquier otro comando de
shell se deniega sin preguntar. Si una tarea debe compilar, probar o pasar
lint, usa `--allow-shell <herramienta>[,<herramienta>]` — o pide el comando
explícitamente en la tarea (p. ej. "ejecuta `dotnet test`"): las herramientas
de build (`dotnet`, `npm`, `ng`, `pytest`, …) se permiten automáticamente; los
intérpretes como `node` o `python` necesitan un `--allow-shell` explícito. Cada
herramienta se convierte en un `--allow-tool='shell(<herramienta>:*)'`. Los
shells de comandos (`powershell`, `cmd`, `bash`…), los ejecutores de comandos y
de privilegios (`env`, `xargs`, `sudo`…), los comandos de borrado y los CLIs de
red/remotos/nube (`gh`, `ssh`, `curl`, `az`…) se rechazan, y nunca se usa
`--allow-all`.

Las reglas de negación solo aplican a los comandos que Copilot ejecuta
directamente. En cuanto se permite cualquier herramienta de shell — con
`--allow-shell` o por autodetección, que se activa solo por la redacción de la
tarea y no verifica si el repo es de confianza — Copilot puede escribir un
script, un script de npm, una receta de Makefile o un target de MSBuild y
ejecutarlo con esa herramienta, y ese código puede borrar archivos, hacer push
o usar la red sin chocar con ninguna regla de negación. Trata cualquier permiso
de shell como ejecución de código completa con tus credenciales, y revisa
`git status`, `git log` y `git reflog` después de la corrida. Los feeds de
paquetes privados siguen necesitando credenciales en el entorno de Copilot: un
401 en el restore es autenticación, no permisos.

## Problemas conocidos del CLI que este plugin mitiga

| Problema | Solución integrada |
|---|---|
| Bucle infinito de autopiloto en tareas bloqueadas externamente ([copilot-cli#2969](https://github.com/github/copilot-cli/issues/2969)) | Se evita `--autopilot` para tareas acotadas; cuando se usa, `--max-autopilot-continues <N>` siempre se fija explícitamente |
| Reanudar después de un límite de velocidad puede colgar | En salida "rate limit" + ~10s de silencio: termina el proceso e informa, nunca esperes; relanza fresco sin `--continue` |
| Un CLI anterior a 1.0.83 rechaza los flags de seguridad/crédito | Reintenta una vez sin `--no-ask-user` y `--max-ai-credits`, y luego actualiza Copilot CLI |
| Copilot CLI 1.0.83 rechaza `--max-ai-credits` menor a 30 y `--effort` con el modelo Auto por defecto | El límite predeterminado es 30 (valores menores de `--credits` se suben a 30); `--effort` solo se reenvía con un `--model` fijado distinto de `auto` |

## Qué hay en el plugin

| Componente | Propósito |
|---|---|
| `agents/copilot-rescue.md` | Agente forwarder delgado — una llamada `copilot -p`, salida devuelta textualmente |
| `/copilot:rescue` | Delega una tarea explícitamente (`--background`, `--wait`, `--model <name>`, `--credits <N>`, `--effort <level>`, `--allow-shell <tool>`) |
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
- GitHub Copilot CLI ≥ **1.0.83** recomendado (`npm install -g @github/copilot`); 1.0.67 sigue siendo el mínimo para `--model`

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
