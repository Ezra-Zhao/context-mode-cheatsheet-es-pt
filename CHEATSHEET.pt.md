# 🧠 Context Mode — Guia rápido (Português)

> Comandos e números verificados no README e no BENCHMARK.md de [mksglu/context-mode](https://github.com/mksglu/context-mode) (2026-10-12).
> O que é: um servidor MCP com hooks que mantém a saída bruta das ferramentas fora da sua janela de contexto. O próprio projeto se define como "a outra metade do problema do contexto".

---

## 0. Por que você precisa

Toda chamada MCP despeja dado bruto na janela de contexto:

- Um snapshot do Playwright: **56 KB**
- Vinte issues do GitHub: **59 KB**
- Um access log: **45 KB**

Em uns 30 minutos, cerca de 40% do contexto já era — e contexto é token, e token é dinheiro. Quando o agente compacta a conversa para liberar espaço, ele também esquece quais arquivos estava editando e o que você pediu por último.

No benchmark do próprio projeto, sessão completa: **315 KB de saída bruta viram 5.4 KB (98% menos)**, e a sessão sai de ~30 minutos para ~3 horas.

---

## 1. Instalação

**Claude Code** (plugin, tudo automático — exige Claude Code v1.0.33+):

```
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```

Reinicie o Claude Code (ou rode `/reload-plugins`) e confira — cada verificação deve mostrar `[x]`:

```
/context-mode:ctx-doctor
```

Teste só com MCP (as 11 ferramentas, sem roteamento automático):

```bash
claude mcp add context-mode -- npx -y context-mode
```

**Outras plataformas** (Node.js ≥ 22.5 ou Bun):

```bash
npm install -g context-mode
```

Depois cadastre o servidor MCP e os hooks conforme a plataforma — o README oficial tem uma seção para cada uma (Gemini CLI: um bloco em `~/.gemini/settings.json`; VS Code Copilot: `.vscode/mcp.json` + `.github/hooks/context-mode.json`; GitHub Copilot CLI: `copilot plugin install mksglu/context-mode:configs/copilot-cli`).

---

## 2. Primeira sessão, 4 passos

1. Instale (acima) e rode `ctx doctor` — ou `/context-mode:ctx-doctor` no Claude Code.
2. Trabalhe normalmente. Os hooks desviam para o sandbox as saídas grandes (Bash, Read, WebFetch, chamadas MCP externas) sem você fazer nada.
3. Indexe os documentos que vai consultar toda hora: `ctx index` uma vez, e depois `ctx search` em vez de reler tudo.
4. Veja o resultado quando quiser: `ctx stats` — detalhe por ferramenta, tokens consumidos, taxa de economia.

No Claude Code existem os comandos slash: `/context-mode:ctx-stats`, `ctx-doctor`, `ctx-index`, `ctx-search`, `ctx-upgrade`, `ctx-purge`, `ctx-insight`. Nas demais plataformas, digite `ctx stats`, `ctx doctor` etc. no chat.

Pelo terminal, sem sessão de IA:

```bash
context-mode doctor
context-mode index . --source project:my-app
context-mode search "authentication middleware" --source project:my-app
context-mode upgrade
```

---

## 3. As 11 ferramentas MCP

Seis de sandbox fazem o trabalho; cinco de gestão cuidam da instalação.

| Ferramenta | O que faz | Bruto → contexto |
|---|---|---|
| `ctx_batch_execute` | Vários comandos + várias buscas em UMA chamada (`concurrency: 1-8` opcional) | 986 KB → 62 KB |
| `ctx_execute` | Roda código em 12 linguagens; só o stdout entra no contexto | 56 KB → 299 B |
| `ctx_execute_file` | Processa um arquivo dentro do sandbox; o conteúdo bruto não sai | 45 KB → 155 B |
| `ctx_index` | Fatia markdown no FTS5 com ranking BM25 | 60 KB → 40 B |
| `ctx_search` | Consulta o conteúdo indexado, várias consultas por chamada | recuperação sob demanda |
| `ctx_fetch_and_index` | Baixa uma URL, fatia e indexa (cache TTL de 24 h) | 60 KB → 40 B |
| `ctx_stats` | Economia, contagem de chamadas, estatísticas da sessão | — |
| `ctx_doctor` | Diagnostica runtimes, hooks, FTS5, versões | — |
| `ctx_upgrade` | Atualiza, reconstrói, reconfigura os hooks | — |
| `ctx_purge` | Apaga todo o conteúdo indexado | — |
| `ctx_insight` | Painel de analytics para equipes | — |

---

## 4. Como o sandbox funciona

Cada chamada ao `ctx_execute` abre um subprocesso isolado. Seu código roda lá dentro, o stdout é capturado e **só o stdout entra na conversa**. Logs, respostas de API e snapshots nunca saem do sandbox, e um script não enxerga a memória do outro.

- 12 linguagens: JavaScript, TypeScript, Python, Shell, Ruby, Go, Rust, PHP, Perl, R, Elixir, C#. O Bun é detectado sozinho e roda JS/TS de 3 a 5 vezes mais rápido.
- CLIs autenticadas (`gh`, `aws`, `gcloud`, `kubectl`, `docker`) herdam as credenciais por passthrough — a conversa nunca as vê.
- Saída acima de 5 KB com um `intent` definido? Ela é indexada e filtrada: você recebe só os trechos que batem com a intenção, mais termos de busca para continuar perguntando.

A ideia que o projeto defende é **Think in Code**: em vez de ler 50 arquivos para dentro do contexto só para contar funções, você escreve um script que conta e imprime o resultado. No exemplo deles: 47 × Read() = 700 KB viram 1 × `ctx_execute()` = 3.6 KB.

---

## 5. A base de conhecimento

O `ctx_index` fatia o markdown por títulos (blocos de código ficam intactos) numa tabela **SQLite FTS5**. A busca combina duas estratégias unidas por Reciprocal Rank Fusion:

- **Porter stemming** — "caching" também encontra "cached", "caches".
- **Trigramas** — "useEff" encontra "useEffect".

E mais: ranking BM25 com títulos valendo 5×, reordenação por proximidade em consultas com vários termos, correção fuzzy com Levenshtein para erro de digitação ("kuberntes" → "kubernetes") e trechos inteligentes que extraem janelas em volta dos seus termos em vez de cortar o texto.

URLs baixadas ficam em cache (TTL padrão de 24 h, ajustável por chamada; `ttl: 0` ou `force: true` ignora o cache). Conteúdo com mais de 14 dias é apagado na inicialização. Os bancos ficam no seu disco, ex.: `~/.context-mode/content/`.

---

## 6. Continuidade de sessão

Toda edição de arquivo, operação git, tarefa, erro e decisão sua fica registrada no SQLite. Quando a conversa compacta, o context-mode **não** despeja tudo de volta no contexto: ele recupera só os eventos relevantes com busca BM25, e o modelo continua exatamente de onde parou.

Abriu sem `--continue`? Os dados da sessão anterior são apagados na hora: sessão nova, quadro limpo.

Um limite que o projeto declara sem rodeio: o roteamento controla **para onde os dados vão**, nunca como o modelo escreve a resposta. Nada de forçar brevidade.

---

## 7. Roteamento: hooks contra arquivos de instrução

Hooks interceptam as chamadas antes de executar — podem bloquear um comando e desviá-lo para o sandbox. Arquivos de instrução só aconselham o modelo; não bloqueiam nada. Números do próprio projeto:

| Configuração | Contexto economizado |
|---|---|
| Hooks ativos | **~98%** |
| Só arquivo de instrução | ~60% |

Um único snapshot do Playwright sem roteamento (56 KB) apaga a economia de uma sessão inteira. Ative os hooks onde a sua plataforma permitir.

---

## 8. Plataformas

17 clientes compatíveis, mais a integração com o gateway OpenClaw (contagem oficial). Da tabela de compatibilidade:

Claude Code · Qwen Code · Gemini CLI · VS Code Copilot · JetBrains Copilot · GitHub Copilot CLI · Cursor · OpenCode · KiloCode · OpenClaw · Codex CLI · Kimi Code · Antigravity · Antigravity CLI (`agy`) · Kiro · Zed · Pi · OMP

O suporte a hooks varia: Antigravity e Zed não têm hooks e dependem de copiar o arquivo de roteamento na mão; no Codex CLI é preciso ativar `[features].hooks = true`. Matriz completa: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md).

---

## 9. Benchmarks (oficiais, com saídas reais)

| Cenário | Bruto | Contexto | Economia |
|---|---|---|---|
| Snapshot do Playwright | 56.2 KB | 299 B | 99% |
| Issues do GitHub (20) | 58.9 KB | 1.1 KB | 98% |
| Access log (500 requisições) | 45.1 KB | 155 B | 100% |
| CSV de analytics (500 linhas) | 85.5 KB | 222 B | 100% |
| Git log (153 commits) | 11.6 KB | 107 B | 99% |
| Pesquisa em repo (subagente) | 986 KB | 62 KB | 94% |

Série completa: 21 cenários em [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md) — 376 KB brutos → 16.5 KB de contexto (96% no total).

---

## 10. O mesmo problema, outra aposta: billion-context

O [ranxianglei/billion-context](https://github.com/ranxianglei/billion-context) (1.1k ⭐) também é um plugin de compressão de contexto, segundo a descrição do próprio repo. Apostas diferentes:

| | Context Mode | billion-context |
|---|---|---|
| Ideia central | Sandbox na saída + base FTS5; recuperar sob demanda | Comprimir para caber numa janela pequena (100K, segundo a descrição dele) |
| Número de capa | 98% menos contexto na saída das ferramentas | 5× menos tokens (descrição dele) |
| Sessão | Continuidade entre compactações com eventos no SQLite | Sessões únicas de um mês, bilhões de tokens (descrição dele) |
| Alcance | 17 plataformas via MCP + hooks | Plugin pensado para janelas pequenas |

---

## 11. Para fechar

- **Tudo local.** Sem telemetria, sem sincronização em nuvem, sem conta. Código, prompts e dados da sessão ficam na sua máquina.
- **Licença:** o projeto usa Elastic License 2.0 (código-fonte disponível) — grátis para usar, bifurcar e modificar; você não pode oferecê-lo como serviço hospedado/gerenciado.
- **Onde ele guarda os dados** dá para mudar: variável `CONTEXT_MODE_DIR` com caminho absoluto (senão vale o padrão do adaptador, ex.: `~/.claude/context-mode`).

---

*Fonte: https://github.com/mksglu/context-mode · Docs: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md) · Benchmarks: [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md)*
