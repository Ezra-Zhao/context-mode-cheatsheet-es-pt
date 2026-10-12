# 🧠 Context Mode — Guía rápida (Español)

> Comandos y cifras verificados contra el README y BENCHMARK.md de [mksglu/context-mode](https://github.com/mksglu/context-mode) (2026-10-12).
> Qué es: un servidor MCP con hooks que mantiene la salida cruda de las herramientas fuera de tu ventana de contexto. El proyecto lo define como "la otra mitad del problema del contexto".

---

## 0. Por qué lo necesitas

Cada llamada MCP vuelca datos crudos en la ventana de contexto:

- Un snapshot de Playwright: **56 KB**
- Veinte issues de GitHub: **59 KB**
- Un access log: **45 KB**

A los 30 minutos ya perdiste cerca del 40% del contexto — y el contexto son tokens, y los tokens cuestan dinero. Cuando el agente compacta la conversación para liberar espacio, también olvida qué archivos estaba editando y qué le pediste por última vez.

El benchmark del propio proyecto, sesión completa: **315 KB de salida cruda se convierten en 5.4 KB (98% menos)**, y la sesión pasa de ~30 minutos a ~3 horas.

---

## 1. Instalación

**Claude Code** (plugin, totalmente automático — requiere Claude Code v1.0.33+):

```
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```

Reinicia Claude Code (o ejecuta `/reload-plugins`) y verifica — cada comprobación debe salir `[x]`:

```
/context-mode:ctx-doctor
```

Prueba solo MCP (las 11 herramientas, sin enrutado automático):

```bash
claude mcp add context-mode -- npx -y context-mode
```

**Otras plataformas** (Node.js ≥ 22.5 o Bun):

```bash
npm install -g context-mode
```

Después agrega el servidor MCP y los hooks según tu plataforma — el README oficial trae una sección por plataforma (Gemini CLI: un bloque en `~/.gemini/settings.json`; VS Code Copilot: `.vscode/mcp.json` + `.github/hooks/context-mode.json`; GitHub Copilot CLI: `copilot plugin install mksglu/context-mode:configs/copilot-cli`).

---

## 2. Primera sesión, 4 pasos

1. Instala (arriba) y ejecuta `ctx doctor` — o `/context-mode:ctx-doctor` en Claude Code.
2. Trabaja como siempre. Los hooks desvían al sandbox las salidas grandes (Bash, Read, WebFetch, llamadas MCP externas) sin que hagas nada.
3. Indexa los documentos que vas a consultar seguido: `ctx index` una vez, y luego `ctx search` en vez de releer.
4. Mira el resultado cuando quieras: `ctx stats` — detalle por herramienta, tokens consumidos, ratio de ahorro.

En Claude Code hay comandos slash: `/context-mode:ctx-stats`, `ctx-doctor`, `ctx-index`, `ctx-search`, `ctx-upgrade`, `ctx-purge`, `ctx-insight`. En el resto de plataformas escribes `ctx stats`, `ctx doctor`, etc. en el chat.

Desde la terminal, sin sesión de IA:

```bash
context-mode doctor
context-mode index . --source project:my-app
context-mode search "authentication middleware" --source project:my-app
context-mode upgrade
```

---

## 3. Las 11 herramientas MCP

Seis de sandbox hacen el trabajo; cinco de gestión administran la instalación.

| Herramienta | Qué hace | Crudo → contexto |
|---|---|---|
| `ctx_batch_execute` | Muchas órdenes + muchas búsquedas en UNA sola llamada (`concurrency: 1-8` opcional) | 986 KB → 62 KB |
| `ctx_execute` | Ejecuta código en 12 lenguajes; solo stdout entra al contexto | 56 KB → 299 B |
| `ctx_execute_file` | Procesa un archivo dentro del sandbox; el contenido crudo no sale | 45 KB → 155 B |
| `ctx_index` | Trocea markdown en FTS5 con ranking BM25 | 60 KB → 40 B |
| `ctx_search` | Consulta el contenido indexado, varias consultas por llamada | recuperación bajo demanda |
| `ctx_fetch_and_index` | Descarga una URL, la trocea y la indexa (caché TTL de 24 h) | 60 KB → 40 B |
| `ctx_stats` | Ahorro, conteo de llamadas, estadísticas de la sesión | — |
| `ctx_doctor` | Diagnostica runtimes, hooks, FTS5, versiones | — |
| `ctx_upgrade` | Actualiza, reconstruye, reconfigura hooks | — |
| `ctx_purge` | Borra todo el contenido indexado | — |
| `ctx_insight` | Panel de analítica para equipos | — |

---

## 4. Cómo funciona el sandbox

Cada llamada a `ctx_execute` lanza un subproceso aislado. Tu código corre ahí, se captura stdout y **solo stdout entra a la conversación**. Los logs, las respuestas de API y los snapshots nunca salen del sandbox, y ningún script ve la memoria de otro.

- 12 lenguajes: JavaScript, TypeScript, Python, Shell, Ruby, Go, Rust, PHP, Perl, R, Elixir, C#. Bun se detecta solo y ejecuta JS/TS de 3 a 5 veces más rápido.
- Las CLI autenticadas (`gh`, `aws`, `gcloud`, `kubectl`, `docker`) heredan credenciales por passthrough — la conversación nunca las ve.
- ¿Salida mayor de 5 KB y un `intent` definido? Se indexa y se filtra: recibes solo las secciones que coinciden con tu intención, más términos de búsqueda para seguir preguntando.

La idea que empuja el proyecto es **Think in Code**: en vez de leer 50 archivos al contexto para contar funciones, escribes un script que cuenta e imprime el resultado. Su ejemplo: 47 × Read() = 700 KB se vuelve 1 × `ctx_execute()` = 3.6 KB.

---

## 5. La base de conocimiento

`ctx_index` trocea el markdown por encabezados (los bloques de código quedan intactos) en una tabla **SQLite FTS5**. La búsqueda combina dos estrategias fusionadas con Reciprocal Rank Fusion:

- **Porter stemming** — "caching" también encuentra "cached", "caches".
- **Trigramas** — "useEff" encuentra "useEffect".

Suma ranking BM25 con encabezados ponderados 5×, reordenamiento por proximidad en consultas de varios términos, corrección difusa con Levenshtein para errores de tipeo ("kuberntes" → "kubernetes") y fragmentos inteligentes que extraen ventanas alrededor de tus términos en vez de truncar.

Las URL descargadas quedan en caché (TTL por defecto 24 h, ajustable por llamada; `ttl: 0` o `force: true` lo saltan). El contenido de más de 14 días se limpia al arrancar. Las bases quedan en local, p. ej. `~/.context-mode/content/`.

---

## 6. Continuidad de sesión

Cada edición de archivo, operación git, tarea, error y decisión tuya queda registrada en SQLite. Cuando la conversación se compacta, context-mode **no** vuelca todo eso de vuelta al contexto: recupera solo los eventos relevantes con búsqueda BM25 y el modelo sigue justo donde estaba.

Si arrancas sin `--continue`, los datos de la sesión anterior se borran al instante: sesión nueva, pizarra limpia.

Un límite que el proyecto declara sin vueltas: el enrutado controla **a dónde van los datos**, nunca cómo escribe el modelo su respuesta. Nada de forzar brevedad.

---

## 7. Enrutado: hooks contra archivos de instrucciones

Los hooks interceptan las llamadas antes de ejecutarse — pueden bloquear una orden y desviarla al sandbox. Los archivos de instrucciones solo aconsejan al modelo; no bloquean nada. Cifras del propio proyecto:

| Configuración | Contexto ahorrado |
|---|---|
| Hooks activos | **~98%** |
| Solo archivo de instrucciones | ~60% |

Un solo snapshot de Playwright sin enrutar (56 KB) puede borrar el ahorro de toda una sesión. Activa los hooks donde tu plataforma lo permita.

---

## 8. Plataformas

17 clientes compatibles más la integración con el gateway OpenClaw (conteo oficial). De la tabla de compatibilidad:

Claude Code · Qwen Code · Gemini CLI · VS Code Copilot · JetBrains Copilot · GitHub Copilot CLI · Cursor · OpenCode · KiloCode · OpenClaw · Codex CLI · Kimi Code · Antigravity · Antigravity CLI (`agy`) · Kiro · Zed · Pi · OMP

El soporte de hooks varía: Antigravity y Zed no tienen hooks y dependen de copiar el archivo de enrutado a mano; en Codex CLI hay que activar `[features].hooks = true`. Matriz completa: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md).

---

## 9. Benchmarks (oficiales, con salidas reales)

| Escenario | Crudo | Contexto | Ahorro |
|---|---|---|---|
| Snapshot de Playwright | 56.2 KB | 299 B | 99% |
| Issues de GitHub (20) | 58.9 KB | 1.1 KB | 98% |
| Access log (500 peticiones) | 45.1 KB | 155 B | 100% |
| CSV de analítica (500 filas) | 85.5 KB | 222 B | 100% |
| Git log (153 commits) | 11.6 KB | 107 B | 99% |
| Investigación de repo (subagente) | 986 KB | 62 KB | 94% |

Serie completa: 21 escenarios en [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md) — 376 KB crudos → 16.5 KB de contexto (96% en total).

---

## 10. El mismo problema, otra apuesta: billion-context

[ranxianglei/billion-context](https://github.com/ranxianglei/billion-context) (1.1k ⭐) también es un plugin de compresión de contexto, según la descripción de su propio repo. Apuestas distintas:

| | Context Mode | billion-context |
|---|---|---|
| Idea central | Sandbox para la salida + base FTS5; recuperar bajo demanda | Comprimir para que baste una ventana chica (100K, según su descripción) |
| Cifra titular | 98% menos contexto en la salida de herramientas | 5× menos tokens (su descripción) |
| Sesión | Continuidad entre compactaciones con eventos en SQLite | Sesiones únicas de un mes, miles de millones de tokens (su descripción) |
| Alcance | 17 plataformas vía MCP + hooks | Plugin pensado para ventanas chicas |

---

## 11. Para cerrar

- **Todo local.** Sin telemetría, sin sincronización en la nube, sin cuenta. Código, prompts y datos de sesión quedan en tu máquina.
- **Licencia:** el proyecto usa Elastic License 2.0 (código fuente disponible) — gratis para usar, bifurcar y modificar; no puedes ofrecerlo como servicio alojado/gestionado.
- **Dónde guarda los datos** se puede mover: variable `CONTEXT_MODE_DIR` con ruta absoluta (si no, el valor por defecto del adaptador, p. ej. `~/.claude/context-mode`).

---

*Fuente: https://github.com/mksglu/context-mode · Docs: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md) · Benchmarks: [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md)*
