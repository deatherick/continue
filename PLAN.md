# Plan: Continue Fork -- Config Sync integrado + modo Auto/Bypass visual

**Repo:** https://github.com/deatherick/continue (fork de continuedev/continue, Apache 2.0)
**Fecha del plan:** 2026-09-05
**Para quién es esto:** otra sesión/modelo de Claude Code (o cualquier LLM
agente), en una conversación NUEVA abierta en este directorio
(`~/code/continue-fork`). Esta sesión (la que escribió el plan) está
dedicada a comparar modelos LLM locales (Qwen3.6, Phi-4, etc.) y no debe
mezclar su contexto con este trabajo de ingeniería -- por eso el plan vive
en un archivo, no en la conversación.

**Contexto de por qué existe este proyecto:** Erick usa VS Code + Continue
(discontinuado por Cursor en 2026, pero Apache 2.0 y forkeable) con modelos
locales vía Ollama en una Mac Studio, desde varias máquinas. Ya construimos
una extensión separada ("Continue Config Sync",
`~/code/continue-sync-ext/`) que resuelve el sync de `config.yaml` entre
máquinas vía un repo privado de GitHub -- funciona, está probada en 2
máquinas reales. Este plan la **integra directamente en el fork de
Continue** (una sola extensión, no dos), y agrega una segunda feature real:
un toggle visual de "Auto/Bypass" en el mismo selector de modo
Chat/Plan/Agent, al estilo de Claude Code.

No se investigó nada más allá de lo que este documento cita -- cada
afirmación sobre el código de Continue está verificada contra el código
real del repo (`gh api repos/continuedev/continue/contents/...`), no contra
documentación ni resúmenes.

---

## Feature 1: Integrar Continue Sync en esta extensión

### Qué existe ya (para copiar, no reinventar)

`~/code/continue-sync-ext/src/extension.ts` (proyecto separado, ya
funcional) tiene TODA la lógica necesaria:
- Login GitHub vía `vscode.authentication.getSession("github", ["repo"])`.
- Polling con ETag contra `GET /repos/{repo}/contents/{path}`.
- Detección de node/npx locales (`detectNodeBin()`), multi-estrategia
  (which/where, luego rutas conocidas de nvm/Homebrew/Program Files).
- Render de placeholders (`{{API_BASE}}`, `{{NPX_CMD}}`, `{{NODE_CMD}}`,
  `{{PATH_ENV}}`, `{{DATETIME_SCRIPT}}`, `{{GITHUB_TOKEN}}`).
- Preservación del token de GitHub ya puesto en cada máquina (nunca se
  sincroniza un secreto).
- Lock file entre ventanas (`tryAcquireLock`/`releaseLock`) -- necesario:
  varias ventanas de VS Code = varios Extension Hosts = condición de
  carrera real si no hay lock (nos pasó, ver commit history de ese repo).
- Comando "Publish Template to GitHub" (sube cambios locales al repo).
- Validación anti-regresión: rechaza escribir el config si queda algún
  `{{PLACEHOLDER}}` sin resolver (nos salvó de un bug real).

### Dónde va en este repo

- **Punto de entrada de la extensión VS Code:**
  `extensions/vscode/src/extension.ts` (función `activate`). Portar ahí la
  lógica de `continue-sync-ext/src/extension.ts`, adaptada:
  - En vez de escribir `~/.continue/config.yaml` completo (que aquí YA lo
    gestiona Continue mismo), el target correcto es fusionar/mergear
    valores de modelo (`apiBase`, etc.) hacia la config que Continue ya
    carga -- ver `packages/config-yaml` para el loader real. Alternativa
    más simple y de menor riesgo: mantener el mismo diseño (un
    `template.yaml` externo con placeholders que se renderiza a
    `~/.continue/config.yaml`), ya que es exactamente el archivo que este
    mismo Continue lee. Empezar con esta opción -- es la ya probada.
- **Manifiesto** (`extensions/vscode/package.json`): agregar los comandos
  (`continueSync.pullNow`, `.publishTemplate`, `.configureRepo`,
  `.editTemplate`, `.showStatus`) y las properties de configuración
  (`continueSync.repo`, `.pollIntervalSeconds`, `.role` con
  `"scope": "machine"`, `.ollamaHost` con `"scope": "machine"` -- el scope
  machine es CRÍTICO, ver nota abajo) al `contributes` existente, sin
  romper lo que ya declara Continue ahí.
- **Nota de scope crítica (encontrada en producción):** si Erick usa
  Settings Sync de VS Code entre sus máquinas (lo usa), cualquier setting
  SIN `"scope": "machine"` se sincroniza entre ellas -- y `role`/`ollamaHost`
  son genuinamente distintos por máquina (una es "server", otras "client").
  Sin el scope correcto, una máquina le pisa el valor a la otra.
- **Assets a llevar:** copiar `~/local-llm-bench/mcp-servers/datetime-mcp-server.js`
  y el contenido actual de `~/local-llm-bench/continue-sync/config.template.yaml`
  (o la versión ya en https://github.com/deatherick/continue-sync-config,
  que es la fuente de verdad vigente) como referencia de qué debe poder
  renderizar el sync integrado.

### Criterio de aceptación

- Instalar la extensión empaquetada de este fork en una máquina limpia,
  configurar `continueSync.repo`, loguearse con GitHub, y que
  `~/.continue/config.yaml` se genere solo, sin ningún paso manual de
  copiar archivos.
- Confirmar que el token de GitHub puesto a mano en el MCP server "GitHub"
  sobrevive un re-sync (no se sobreescribe con el placeholder).
- Confirmar con 2+ ventanas de VS Code abiertas simultáneamente que no hay
  escritura duplicada/carrera (revisar logs, no debería haber dos
  "Applied config" pisándose en la misma ventana de tiempo).

---

## Feature 2: Modo "Auto" / "Bypass" visual en el selector Chat/Plan/Agent

Objetivo: igual que Claude Code, un modo donde las tool calls se ejecutan
sin preguntar, seleccionable desde la MISMA interfaz donde ya se elige
Chat/Plan/Agent -- no un checkbox escondido en Settings.

### Estado actual verificado (no asumido)

- El selector de modo vive en
  `gui/src/components/ModeSelect/ModeSelect.tsx` -- un `Listbox` con 3
  opciones (`chat`/`plan`/`agent`), despacha `setMode(newMode)` de
  `gui/src/redux/slices/sessionSlice.ts`. El tipo `MessageModes` (de
  `core`) hoy es `"chat" | "plan" | "agent"`.
- La aprobación de tool calls vive en
  `gui/src/redux/thunks/evaluateToolPolicies.ts`, función
  `evaluateToolPolicy`: resuelve `basePolicy` desde
  `toolPolicies[toolName] ?? tool.defaultToolPolicy ?? DEFAULT_TOOL_SETTING`
  (`DEFAULT_TOOL_SETTING = "allowedWithPermission"`, definido en
  `gui/src/redux/slices/uiSlice.ts`).
- El estado persistente de políticas por-tool vive en
  `uiSlice.ts` → `UIState.toolSettings: ToolPolicies` (`{[toolName]: ToolPolicy}`),
  con reducers ya existentes (`addTool`, etc. -- revisar el resto del
  archivo, se cortó en la lectura a los ~80 líneas).
- **Confirmado que NO existe hoy** ningún campo de política en el schema
  real de `config.yaml` para MCP servers
  (`packages/config-yaml/src/schemas/mcp/index.ts` -- leído completo, no
  tiene `policy` ni `defaultToolPolicy`, solo `name`, `command`/`url`,
  `args`, `env`, `cwd`, `type`, `apiKey`, `requestOptions`,
  `connectionTimeout`). El campo `defaultToolPolicy` SÍ existe en el tipo
  interno `Tool` (`core/index.d.ts` línea ~1154) pero solo lo usan las
  tools *built-in* (hardcodeado en cada `core/tools/definitions/*.ts`) --
  las tools de servidores MCP nunca lo reciben.
- El punto exacto donde un servidor MCP se convierte en `Tool[]` es
  `core/config/profile/doLoadConfig.ts`, dentro del bloque
  `if (server.status === "connected") { const serverTools: Tool[] = server.tools.map(...) }`
  (línea ~211). Ahí es donde habría que agregar
  `defaultToolPolicy: server.defaultToolPolicy` si se quiere que MCP
  servers declaren su propio default vía config.yaml (mejora relacionada
  pero NO necesaria para el modo Auto -- el modo Auto de abajo es un
  override global, no depende de esto).

### Diseño propuesto para el modo Auto

**No** convertir "Auto" en un cuarto valor de `MessageModes` (evitar tocar
el enum en 20+ archivos que hacen `mode === "agent"`). En vez de eso: un
flag ortogonal, igual a como Claude Code lo maneja (el modo Plan/Agent es
una cosa, "bypass permissions" es otra).

1. **`gui/src/redux/slices/uiSlice.ts`**: agregar
   `autoApprovalMode: boolean` a `UIState` (default `false`), y un reducer
   `setAutoApprovalMode(state, action: PayloadAction<boolean>)`.
2. **`gui/src/redux/thunks/evaluateToolPolicies.ts`**: en
   `evaluateToolPolicy`, ANTES de calcular `basePolicy`, si
   `store.ui.autoApprovalMode === true` Y el tool no está explícitamente en
   `disabled` (revisar esa exclusión primero, nunca se debe auto-aprobar
   algo que el usuario apagó a propósito), retornar
   `{ policy: "allowedWithoutPermission", toolCallState }` de una vez,
   saltándose el resto de la lógica de policy dinámica.
3. **`gui/src/components/ModeSelect/ModeSelect.tsx`**: agregar una 4ta
   `ListboxOption` "Auto" (ícono sugerido: un rayo/bolt, consistente con
   `ModeIcon.tsx` -- revisar ese archivo para el patrón de íconos por
   modo). Al seleccionarla: `dispatch(setMode("agent"))` +
   `dispatch(setAutoApprovalMode(true))` en el mismo `selectMode`/handler
   (agent mode es prerequisito: sin tools activas no hay nada que
   auto-aprobar). Al seleccionar cualquier OTRA opción, resetear
   `autoApprovalMode` a `false`.
4. **Indicador visual** (importante, que sea obvio que estás en modo
   peligroso): cuando `autoApprovalMode` esté activo, el botón del
   `ListboxButton` debería mostrar un color/borde distinto (ej. ámbar/rojo
   sutil) -- mismo principio que Claude Code muestra en su UI de terminal
   cuando bypass permissions está activo. Referencia de paleta: revisar
   `gui/src/components/ModeSelect/ModeIcon.tsx` y las clases Tailwind ya
   usadas en `ModeSelect.tsx` (`text-warning` ya existe, usarlo).
5. **Persistencia entre sesiones**: decidir si `autoApprovalMode` debe
   resetearse a `false` en cada sesión nueva (recomendado, más seguro -- que
   sea un modo que se active a propósito cada vez, no algo que quede
   pegado sin que el usuario se acuerde) o persistir. Si se persiste,
   sería vía el mismo mecanismo de redux-persist que ya usa `uiSlice`
   (revisar `gui/src/redux/store.ts` o similar para el persistConfig
   actual).

### Criterio de aceptación

- Seleccionar "Auto" en el dropdown ejecuta tool calls (incluyendo las de
  nuestros MCP servers: Web Search, Git, Context7, Date&Time) sin mostrar
  el diálogo de permiso.
- Un tool marcado `disabled` explícitamente por el usuario sigue sin
  ejecutarse aunque "Auto" esté activo (verificar que el check de
  `disabled` corre ANTES del bypass, no después).
- Cambiar a "Chat" o "Plan" desde "Auto" apaga el auto-approve (confirmar
  que no queda una tool call en curso ejecutándose sin permiso de forma
  colgada).
- El indicador visual dejacla evidente que Auto está activo (captura de
  pantalla antes/después en la PR o en el mensaje de handoff).

---

## Cómo empezar (para la sesión que implemente esto)

```bash
cd ~/code/continue-fork
git remote -v   # confirmar: origin=deatherick/continue, upstream=continuedev/continue
```

1. Leer este archivo completo antes de tocar código.
2. Instalar dependencias del monorepo (es pnpm workspaces -- revisar
   `package.json` raíz y `pnpm-lock.yaml` para la versión exacta de pnpm
   requerida antes de correr `pnpm install`).
3. Empezar por Feature 2 (más chica, más aislada, sirve para validar que
   el pipeline de build/test del monorepo funciona antes de meterse con
   Feature 1 que es más grande).
4. Para Feature 1, tener abierto en paralelo
   `~/code/continue-sync-ext/src/extension.ts` como referencia -- es
   código ya probado, no reinventar la lógica, solo trasplantarla.
5. Empaquetar (`extensions/vscode`, revisar su `package.json` para el
   script de build/package exacto de este monorepo -- probablemente
   distinto al `vsce package` simple que usamos en el proyecto standalone,
   por el monorepo/workspace) y probar instalando el `.vsix` resultante
   igual que hicimos con `continue-sync-ext` (`code --install-extension`).
6. No hace falta abrir PR contra `continuedev/continue` (upstream) --
   este es un fork personal para uso propio, no necesariamente para
   contribuir de vuelta (aunque nada lo impide, Apache 2.0 permite ambos).

## Cosas explícitamente fuera de alcance de este plan

- No migrar el Continue Hub / cloud sync (está muerto, no se puede
  revivir, no es el objetivo -- el objetivo es la sync propia vía GitHub).
- No tocar la lógica de terminal-security / built-in tools existentes más
  allá de lo estrictamente necesario para el bypass de Feature 2.
- No publicar esta extensión al Marketplace -- uso personal, instalación
  vía `.vsix` como ya se viene haciendo.
