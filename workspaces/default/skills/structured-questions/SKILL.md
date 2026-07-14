---
name: structured-questions
description: "Ask the user structured questions with clickable buttons via the BMO server MCP. Use create_question to render a rich question card in the chat — the user clicks an option instead of typing. Handles natural-language choice (answer_mode natural) and raw machine routing (answer_mode raw)."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "❓"
---

# Structured Questions (Perguntas com Botões)

## Vocabulary PT↔EN

| Português | English (tool field) |
|-----------|----------------------|
| pergunta / questão | question |
| opção / alternativa | option |
| botão | button |
| modo de resposta | answer_mode |
| natural (conversa) | natural |
| bruto / máquina | raw |
| roteamento | routing |
| destino | sink |
| bloco rico | rich block |
| cartão / card | rich content card / widget |

The user speaks Portuguese; all MCP tool names and fields are in English — map accordingly.

---

## Purpose

When the BMO needs the user to choose between options, use `create_question` to render a **rich question card** with clickable buttons in the chat. The user clicks instead of typing — faster, unambiguous, and the frontend handles the routing automatically.

**Use this when:**
- The user needs to pick from 2–4 concrete options ("prefere resumo ou detalhes?")
- The choice should be routed to a machine setting without the BMO reinterpreting ("qual modelo de imagem usar?")
- The answer needs to be unambiguous (button click → exact value, no parsing)

**Don't use this when:**
- The question is open-ended ("o que você acha?") — just ask in plain text
- There's only one obvious path forward — just do it
- The user is already mid-conversation about something else — a plain text question is less disruptive

---

## Available Tools

| Tool | When to use |
|------|-------------|
| `create_question` | Ask the user a question with clickable buttons. Returns a `rich_block` ready to emit. |
| `get_question` | Look up a past question by id — use when the user asks about a previous question ("aquela pergunta que eu respondi antes"). |

---

## answer_mode: `natural` vs `raw`

This is the central decision. Pick the wrong mode and the answer goes to the wrong place.

### `natural` — conversational choice

The BMO asks a question on its own initiative and needs the answer to continue the conversation. When the user clicks a button, the `value` is injected as a **normal chat message** from the user — as if they typed it. The BMO sees it in the next turn and responds naturally.

**Use when:**
- The BMO is offering options as part of a task ("prefere um resumo curto ou os detalhes completos?")
- The answer determines what the BMO does next ("quer que eu pesquise em português ou inglês?")
- The choice is conversational and the BMO will act on it

**Routing:**
```json
{"sink": "natural", "session_id": "<sua session ID atual>"}
```

**How to get your session ID:** Look for `Session ID:` in the environment context at the top of your system prompt. Copy the value exactly.

**Example values for natural mode:**
```json
{"options": [
  {"label": "Resumo curto", "value": "summary"},
  {"label": "Detalhes completos", "value": "detail"}
]}
```

### `raw` — machine routing

The answer needs to go intact to a backend destination. The value is routed directly by the server — **the BMO does not see the answer** (unless the backend echoes it back as a side effect).

**Use when:**
- The user is configuring a setting ("qual modelo padrão para imagens?")
- The choice is a machine parameter, not conversational input
- The BMO doesn't need to interpret or act on the answer

**Routing — setting example:**
```json
{"sink": "setting", "key": "image.default_model"}
```

**Example values for raw mode:**
```json
{"options": [
  {"label": "FLUX.2 [klein]", "value": "flux-klein"},
  {"label": "FLUX.2 [schnell]", "value": "flux-schnell"}
]}
```

### Decision flowchart

```
BMO wants user to pick an option
│
├─ BMO needs the answer to continue the conversation?
│   → natural, routing: {"sink":"natural","session_id":"<current>"}
│
└─ Answer goes to a machine/config destination?
    → raw, routing: {"sink":"setting","key":"<setting.key>"}
```

---

## Options Format

Each option is a `{"label": …, "value": …}` pair:

| Field | Purpose | Rules |
|-------|---------|-------|
| `label` | Visible text on the button (human-facing) | Clear, concise, in the user's language. Max ~30 chars. |
| `value` | Identifier sent when clicked (machine-facing) | Short, stable, no spaces, kebab-case. Examples: `"summary"`, `"flux-schnell"`, `"pt-br"`. |

**Good options:**
```json
[
  {"label": "Resumo curto", "value": "summary"},
  {"label": "Detalhes completos", "value": "detail"}
]
```

**Bad options:**
```json
[
  {"label": "opção 1", "value": "opt1"},           // labels should describe the choice
  {"label": "Sim", "value": "Sim, por favor!"}      // value has spaces and punctuation
]
```

---

## Rich Content Delivery (Entrega via Cartão)

The `create_question` tool returns a `QuestionRead` object that includes a **`rich_block`** field — a complete, ready-to-emit `bmo:rich` envelope. The envelope format is documented in AGENTS.md ([Rich Content section](#)). Do **not** reconstruct it.

### Workflow

```
create_question → extract rich_block → emit ```bmo:rich fence → short context phrase → end turn
```

### Step by step

1. **Call `create_question`** with `prompt`, `options`, `answer_mode`, and `routing`.
2. **Extract `rich_block`** from the response. It looks like:
   ```json
   {"v":1,"type":"question","block_id":"question-5","payload":{"question_id":5,"prompt":"...","options":[...],"answer_mode":"natural","status":"pending"},"mutable":true}
   ```
3. **Emit the `rich_block` verbatim** as a `bmo:rich` fenced code block. Serialize it exactly as returned — do not rename fields, do not reconstruct the JSON, do not edit `block_id`.
4. **Add a short context phrase** if natural ("Qual você prefere?") and **end the turn**.

### Example emission

````
```bmo:rich
{"v":1,"type":"question","block_id":"question-5","payload":{"question_id":5,"prompt":"Como você prefere receber as notícias?","options":[{"label":"Resumo curto","value":"summary"},{"label":"Detalhes completos","value":"detail"}],"answer_mode":"natural","status":"pending"},"mutable":true}
```
````

### What the card does

Once the block is in the chat, the frontend:
1. Renders the question with buttons for each option
2. When the user clicks a button, the backend handles routing automatically (natural → chat message; raw → machine destination)
3. The card updates status from `pending` → `answered` via SSE `rich.update` events

**You don't need to do anything else.** Your turn ends after emitting the block.

---

## CRITICAL RULES

### 1. NEVER echo raw tool output

`create_question` returns a full `QuestionRead` object with `id`, `routing`, `block_id`, `rich_block`, and other fields. **Only the `rich_block` goes to the user.** Never dump the raw response in the chat.

- ✅ Extract `rich_block` → emit fence → end turn
- ❌ Print `id`, `routing`, or any other field in the visible text
- ❌ Show the JSON response as text ("A API retornou: ...")

### 2. NEVER leak routing info

The `routing` field is **internal plumbing**. It contains `sink`, `session_id`, and setting keys that are meaningless (or confusing) to the user. These must never appear in visible text.

- ✅ Routing goes only into the `create_question` call parameters
- ❌ "Vou rotear sua resposta para o sink natural na session abc123..."
- ❌ "Routing configurado: sink=setting, key=image.default_model"

### 3. Emit the rich_block verbatim

The `rich_block` field from `create_question` is the **exact** envelope the frontend expects. Do not:
- Reconstruct the JSON yourself
- Rename fields (e.g., don't change `block_id` to `id`)
- Add or remove fields
- Change `mutable` from `true` to `false`

Just `JSON.stringify` the `rich_block` dict and wrap it in the fence.

### 4. End the turn after emitting

Question delivery is fire-and-forget. Do not:
- ❌ Call `get_question` in a loop to check if the user answered
- ❌ Stay in the turn waiting for a response
- ❌ Poll for the answer

The answer arrives in a future turn (natural mode) or goes to the backend (raw mode). Either way, your job is done after the fence.

---

## Workflow Patterns

### Ask a natural-language question

1. **Decide the options.** What are the 2–4 concrete choices? Give each a clear `label` (what the user sees) and a stable `value` (what comes back).
2. **Get your session ID.** Read `Session ID:` from your environment context (at the top of your system prompt).
3. **Call `create_question`:**
   - `prompt`: The question text, in the user's language
   - `options`: Array of `{label, value}` pairs
   - `answer_mode`: `"natural"`
   - `routing`: `{"sink": "natural", "session_id": "<your session ID>"}`
4. **Emit `rich_block`** as a `bmo:rich` fence.
5. **Add a short context phrase** and **end the turn**.

### Ask a raw/config question

1. **Decide the options.** Same as above — clear labels, stable values.
2. **Determine the routing key.** What setting or destination should receive the value? (e.g., `"image.default_model"`)
3. **Call `create_question`:**
   - `prompt`: The question text
   - `options`: Array of `{label, value}` pairs
   - `answer_mode`: `"raw"`
   - `routing`: `{"sink": "setting", "key": "<setting.key>"}`
4. **Emit `rich_block`** as a `bmo:rich` fence.
5. **Add a short context phrase** and **end the turn**.

---

## Flow Examples

### Example 1: Natural mode — conversational choice

**User:** "me dá um resumo das notícias de hoje"

**BMO (turn 1):**
> Claro! Como você prefere receber?
>
> ```bmo:rich
> {"v":1,"type":"question","block_id":"question-12","payload":{"question_id":12,"prompt":"Como você prefere receber as notícias?","options":[{"label":"Resumo curto (1 frase por notícia)","value":"summary"},{"label":"Detalhes completos (2-3 frases)","value":"detail"}],"answer_mode":"natural","status":"pending"},"mutable":true}
> ```

*[Turn ends. User clicks "Resumo curto". The value `"summary"` is injected as a chat message from the user.]*

**User (turn 2, via button click):** `summary`

**BMO (turn 2):** "Aqui vai o resumo curto: [notícias resumidas...]"

### Example 2: Raw mode — config question

**User:** "muda o modelo padrão de imagem"

**BMO (turn 1):**
> Qual modelo você quer como padrão para geração de imagens?
>
> ```bmo:rich
> {"v":1,"type":"question","block_id":"question-15","payload":{"question_id":15,"prompt":"Qual modelo padrão para geração de imagens?","options":[{"label":"FLUX.2 [klein] (rápido, padrão atual)","value":"flux-klein"},{"label":"FLUX.2 [schnell] (ultra-rápido)","value":"flux-schnell"}],"answer_mode":"raw","status":"pending"},"mutable":true}
> ```

*[Turn ends. User clicks "FLUX.2 [schnell]". The value `"flux-schnell"` is routed directly to the `image.default_model` setting. The BMO may not see this answer.]*

### Example 3: Natural mode — binary confirmation

**User:** "limpa as missões concluídas essa semana"

**BMO (turn 1):**
> Você tem 12 missões concluídas essa semana. Quer que eu delete todas?
>
> ```bmo:rich
> {"v":1,"type":"question","block_id":"question-20","payload":{"question_id":20,"prompt":"Deletar 12 missões concluídas dessa semana?","options":[{"label":"Sim, deletar todas","value":"yes"},{"label":"Não, cancelar","value":"no"}],"answer_mode":"natural","status":"pending"},"mutable":true}
> ```

*[Turn ends. User clicks "Sim, deletar todas".]*

**User (turn 2, via button click):** `yes`

**BMO (turn 2):** "Deletando as 12 missões concluídas... [executa a ação]"

---

## Error Handling

| Scenario | How to handle |
|----------|---------------|
| `create_question` fails with validation error | Check that `options` has 2–4 items, each with valid `label` (string) and `value` (string). Check that `answer_mode` is `"natural"` or `"raw"`. |
| User says "não apareceu botão" / "o cartão não carregou" | The frontend may not support `bmo:rich` on this channel. Fall back to asking the question in plain text with enumerated options ("1. Resumo curto, 2. Detalhes completos — responda com o número"). |
| User ignores the buttons and types a response instead | Treat the typed response as the answer. The question card and typed reply are both valid input — don't ask again with buttons. |
| User asks about a past question | Call `get_question` with the question id to check its status and the answer (if any). |
| `question_not_found` on `get_question` | The question may have been deleted or the id is wrong. Tell the user you couldn't find it. |

---

## CRITICAL — Session ID

Your session ID is available in your environment context at the top of every turn:

```
- Session ID: <seu-session-id-aqui>
```

Use this exact value in the `routing` for `natural` mode. Do not guess, hardcode, or reuse a session ID from a past conversation.

If you cannot find `Session ID:` in your context (rare, but possible on some channels), fall back to asking the question in plain text with enumerated options.
