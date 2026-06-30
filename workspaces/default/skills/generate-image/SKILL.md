---
name: generate-image
description: "Generate images via the BMO server MCP image tools (create_image, get_image, list_images). Use whenever the user wants to generate, create, or draw an image — whether from a vague idea or a detailed prompt. Handles prompt enrichment, async lifecycle (pending→generating→done), disciplined waiting, and file delivery."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "🎨"
---

# Generate Image (Geração de Imagens)

## Vocabulary PT↔EN

| Português | English (tool field) |
|-----------|----------------------|
| gerar / criar imagem | create image / generate image |
| imagem / figura / foto | image |
| prompt / descrição | prompt |
| modelo | model |
| estilo | style |
| aguardar / esperar | wait / poll |
| entregar / enviar | deliver / send |
| falhou | failed |
| caminho do arquivo | output_path |

The user speaks Portuguese; all MCP tool names and fields are in English — map accordingly.

---

## Inference Defaults — DO NOT ask for info you can infer

When the user asks to generate an image, infer missing details instead of asking:

- **Vague or short prompt** ("um cachorro astronauta", "uma paisagem bonita"): **do not ask for more detail**. Expand it yourself into a rich, descriptive English prompt following the [Prompt Enrichment](#prompt-enrichment-enriquecimento-de-prompt) rules below. The user wants you to do the creative work.
- **No model mentioned**: omit the `model` parameter entirely. The backend uses whatever is loaded in Draw Things. Do not ask "qual modelo?".
- **No style mentioned**: choose a natural descriptive style that fits the subject. Do not ask.
- **No size/ratio mentioned**: omit `width` and `height` — the backend uses sensible defaults. Do not ask.

### When to ask

Only ask the user when:

- The request is genuinely impossible to interpret (e.g., "faz aquela imagem" with no prior context)
- The user explicitly asks for help choosing between options ("qual modelo você recomenda?")
- The user asks a direct question about capabilities

---

## Available Tools

| Tool | When to use |
|------|-------------|
| `create_image` | Start a new image generation. Returns immediately with `status: "pending"` and an `id`. Does NOT wait for completion. |
| `get_image` | Check the status of a generation and retrieve the result. Returns `status`, and when done, `output_path` (absolute path to the image file on disk). |
| `list_images` | List previously generated images. Use when the user asks about their image history. |

---

## Prompt Enrichment (Enriquecimento de Prompt)

The user almost always gives short, vague requests in Portuguese. You MUST expand these into rich, descriptive English prompts before calling `create_image`. This is the single most important thing you can do to get good results — the model can only work with what you give it.

### FLUX Family (default: FLUX.2 klein)

FLUX models understand **natural, flowing descriptive language**. Write complete sentences, like you're describing a scene to a painter.

**Style:** prose, descriptive, atmospheric. Use adjectives, lighting descriptions, composition hints, mood words.

**Examples:**

| User said | Enriched prompt (send to `create_image`) |
|-----------|------------------------------------------|
| "um cachorro astronauta" | "A golden retriever wearing a detailed white NASA spacesuit with a reflective visor helmet, floating in outer space with Earth visible in the background, stars and nebula clouds, cinematic lighting, highly detailed, 8k resolution" |
| "uma casa na floresta" | "A cozy wooden cabin nestled in a dense misty pine forest at dawn, warm yellow light glowing from the windows, a small stream running nearby, moss-covered stones, soft fog between the trees, atmospheric and peaceful, photorealistic" |

### SDXL Family (Juggernaut, RealVisXL, etc.)

SDXL models prefer **comma-separated tags and short descriptors** — like keyword annotations, not prose.

**Style:** tag-based, concise. Use photography/cinematography jargon. No full sentences.

**Examples:**

| User said | Enriched prompt (send to `create_image`) |
|-----------|------------------------------------------|
| "um cachorro astronauta" | "golden retriever, astronaut suit, NASA helmet, reflective visor, floating in space, earth background, nebula, cinematic lighting, highly detailed, 8k, photorealistic" |
| "retrato de uma mulher" | "portrait of a woman, 35mm, golden hour, natural light, bokeh, detailed skin texture, sharp focus, professional photography, canon r5, 85mm lens" |

### Which style to use

- **FLUX.2 (default), FLUX.1, or any FLUX variant** → natural prose (first table)
- **Juggernaut, RealVisXL, DreamShaper, or any SDXL variant** → comma-separated tags (second table)
- **Uncertain / user didn't specify** → default to natural prose (FLUX style), since FLUX.2 is the default model. Do not ask.

---

## Async Lifecycle

Image generation is **asynchronous and slow** (~90 seconds). Understanding the lifecycle is critical to avoid wasteful polling.

```
pending ──→ generating ──→ done
                │
                └──→ failed
```

| Status | Meaning | What you should do |
|--------|---------|--------------------|
| `pending` | Queued, hasn't started yet | Wait. Do NOT poll. |
| `generating` | Model is actively running | Wait. Do NOT poll (unless well past the ~90s mark). |
| `done` | Image is ready | Call `get_image` to retrieve the `output_path`, then deliver to user. |
| `failed` | Something went wrong | Read `error` field, inform the user honestly. |

`create_image` always returns `status: "pending"` (or occasionally `"generating"`) — it NEVER returns `"done"` on the first call. The generation runs on the Mac mini's GPU via Draw Things.

---

## Waiting Strategy (Estratégia de Espera)

**This is critical.** Every call to `get_image` costs tokens. Polling too early or too often wastes the user's resources with zero benefit — the image won't be ready.

### The disciplined approach

1. **Call `create_image`** with the enriched prompt.
2. **Respond to the user immediately** that generation has started and takes ~90 segundos. Do NOT call `get_image` right after.
3. **Wait ~90 seconds** before the first `get_image` call. There is no point checking sooner — the image will not be `done` in the first 60-90 seconds.
4. **Call `get_image` once** at the ~90s mark:
   - If `done` → deliver the image (go to [Delivering the Image](#delivering-the-image)).
   - If still `generating` or `pending` → now you may enter a **spaced polling loop**: call `get_image` every **15-20 seconds** until `done` or `failed`. Never poll faster than 15s.
   - If `failed` → go to [Handling Failed](#handling-failed).

### Visual summary

```
create_image ──→ tell user "~90s" ──→ wait 90s ──→ get_image
                                                       │
                                          ┌─ done ─────┤
                                          │             │
                                          │    generating/pending
                                          │             │
                                          ▼             ▼
                                   send_file_to_user   wait 15-20s ──→ get_image
                                                                           │
                                                              ┌─ done ─────┤
                                                              │             │
                                                              │    still not done
                                                              │             │
                                                              ▼             ▼
                                                       send_file_to_user   loop (15-20s interval)
```

### Anti-patterns (NEVER do these)

- ❌ Calling `get_image` immediately after `create_image` (image just entered the queue)
- ❌ Polling every 2-5 seconds (burns tokens for no reason — image generation is slow)
- ❌ Calling `get_image` 10+ times in a single turn
- ❌ Staying silent after `create_image` — tell the user what's happening

---

## Delivering the Image

When `get_image` returns `status: "done"`, the response includes an `output_path` field with the absolute path to the image file on the Mac mini's disk.

Example `get_image` response when done:

```json
{
  "id": 42,
  "status": "done",
  "prompt": "A golden retriever wearing a detailed white NASA spacesuit...",
  "model": "FLUX.2 [klein]",
  "output_path": "/Users/jedhai/Library/Application Support/BMO/images/3.png",
  "created_at": "2026-06-30T14:32:00",
  "completed_at": "2026-06-30T14:33:35"
}
```

### The ONLY correct delivery path

```
get_image (done) → send_file_to_user(file_path=output_path)
```

Use the native QwenPaw tool `send_file_to_user` with the `output_path` value exactly as returned. Nothing else.

### 🚫 FORBIDDEN: Shell Access for Images

**Under no circumstances** may you use `execute_shell_command`, `Bash`, or any shell/terminal access to:

- Read, open, preview, or display the image file
- Move, copy, rename, or convert the image file
- Check if the file exists (`ls`, `stat`, `file`, etc.)
- Run any image processing command (`sips`, `magick`, `ffmpeg`, etc.)
- Open the image in Preview or any GUI app (`open`, `qlmanage`, etc.)

**Why:** Shell access bypasses the tool architecture, is not traceable in the MCP audit log, and breaks the contract between BMO and QwenPaw. The correct path is `get_image` → `send_file_to_user`. Always.

If you're tempted to check "does the file exist?" — you already have `output_path` from `get_image` with `status: "done"`. The file exists. Trust the tool.

---

## Honestidade Sobre Visão (Vision Honesty)

**You (DeepSeek) do not have vision capabilities.** You cannot see, view, analyze, or evaluate the generated image. You only know:
- The prompt you sent
- The status (`done` / `failed`)
- The `output_path` (a file path string)
- The model name used

### What this means in practice

- ❌ **NEVER claim you evaluated the image**: "Ficou linda!", "O cachorro ficou perfeito!", "As cores estão ótimas!" — você NÃO viu a imagem. Mentir corrói a confiança do usuário.
- ✅ **Describe what you asked for**: "Aqui está a imagem que eu gerei com o prompt: 'A golden retriever in a NASA spacesuit...' — o prompt descreve um golden retriever com traje espacial flutuando no espaço."
- ✅ **Offer to iterate**: "Se não ficou como você queria, me diz o que ajustar no prompt que eu refaço."
- ✅ **Defer judgment to the user**: "Você é os olhos — me conta se gostou ou quer mudar algo."

The user is the only one who can see the image. Your job is to be a skilled prompt engineer, not an art critic.

---

## Handling Failed

When `get_image` returns `status: "failed"`, the response includes an `error` field with a human-readable message.

Example failure responses:

```json
{
  "id": 43,
  "status": "failed",
  "error": "Draw Things is not running. Please open Draw Things on the Mac mini and ensure a model is loaded."
}
```

```json
{
  "id": 44,
  "status": "failed",
  "error": "Model 'SDXL-RealVis' not found. Available models: FLUX.2 [klein], Juggernaut XL"
}
```

### How to handle failures

1. **Read the `error` field** — it contains actionable information.
2. **Tell the user honestly**: "A geração falhou. Erro: Draw Things não está rodando. Precisa abrir o Draw Things no Mac mini."
3. **Suggest a fix** if the error message points to one (e.g., wrong model name → suggest available models; Draw Things offline → tell user to open it).
4. **Never pretend it succeeded.** Do not say "a imagem ficou pronta" or try to deliver a file that doesn't exist.

---

## Model Selection

- **Default behavior:** Omit the `model` parameter entirely. The backend uses whatever model is currently loaded in Draw Things (typically FLUX.2 [klein]).
- **User explicitly requests a model** ("usa o Juggernaut", "gera com SDXL"): pass the `model` parameter with the name the user gave. If the model name doesn't match what's available, the server will return `failed` with a list of available models — relay that to the user.
- **Do not ask** "qual modelo você quer usar?" — the default omitting behavior is the right answer 90% of the time.

---

## Workflow Patterns

### Generate an image from a user request

1. **Read the user's request.** It will likely be short, vague, and in Portuguese.
2. **Enrich the prompt** following [Prompt Enrichment](#prompt-enrichment-enriquecimento-de-prompt). Expand into a rich English description. Choose prose or tags based on the target model family.
3. **Call `create_image`** with the enriched prompt (and `model` only if the user explicitly asked for one). Do NOT pass `model` by default.
4. **Tell the user immediately**: "Gerando! Isso leva ~90 segundos. [breve descrição do prompt que você montou]."
5. **Wait ~90 seconds** before the first status check.
6. **Call `get_image`** with the image `id`:
   - `done` → call `send_file_to_user(file_path=output_path)`. Describe what you prompted, not what you "see."
   - `generating` / `pending` → tell user "Ainda gerando, vou checar de novo em 15s." Wait 15-20s, then `get_image` again. Loop until `done` or `failed`.
   - `failed` → read `error`, tell user honestly, suggest fix if possible.
7. **After delivery**, offer to adjust: "Se quiser mudar algo no prompt, é só pedir."

### List previous images

1. Call `list_images`. No special parameters needed.
2. Present as a list with ID, prompt (truncated if long), status, and date.

---

## Flow Examples

### Example 1: Complete interaction (happy path)

**User:** "faz uma imagem de um gato samurai"

**BMO (turn 1 — enrichment + create):**
> Vou gerar! Expandi seu pedido para um prompt descritivo em inglês: *"A majestic tabby cat dressed in ornate samurai armor with a kabuto helmet, standing in a bamboo forest at sunset, cherry blossom petals falling, cinematic lighting, highly detailed, 8k, photorealistic"*. A geração leva ~90 segundos.

*[BMO calls: `create_image(prompt="A majestic tabby cat dressed in ornate samurai armor with a kabuto helmet, standing in a bamboo forest at sunset, cherry blossom petals falling, cinematic lighting, highly detailed, 8k, photorealistic")`]*

*[BMO waits 90 seconds]*

**BMO (turn 2 — check):**
*[BMO calls: `get_image(id=45)` → returns `status: "done"`, `output_path: "/Users/jedhai/Library/Application Support/BMO/images/5.png"`]*

> Aqui está! Gerei com o prompt: *"A majestic tabby cat dressed in ornate samurai armor..."*. Se não ficou como você imaginou, me diz que eu ajusto.

*[BMO calls: `send_file_to_user(file_path="/Users/jedhai/Library/Application Support/BMO/images/5.png")`]*

### Example 2: Slow generation (polling loop kicks in)

**User:** "cria uma paisagem cyberpunk"

**BMO (turn 1):**
> Gerando! Prompt: *"A sprawling cyberpunk cityscape at night, neon lights reflecting in rain-slicked streets, flying cars, holographic billboards, towering skyscrapers, Blade Runner aesthetic, cinematic, 8k"*. ~90 segundos.

*[BMO calls: `create_image(prompt="...")` → returns `id: 46`]*
*[BMO waits 90s, calls `get_image(id=46)` → returns `status: "generating"`]*

> Ainda gerando — o modelo está processando. Vou checar de novo em 15s.

*[BMO waits 20s, calls `get_image(id=46)` → returns `status: "done"`, `output_path: ".../6.png"`]*

> Pronto! Aqui está a paisagem cyberpunk. Se quiser ajustar algo (mais chuva, mais neon, outra hora do dia), é só falar.

*[BMO calls: `send_file_to_user(file_path=".../6.png")`]*

### Example 3: Failure

**User:** "gera uma imagem com o modelo DreamShaper"

**BMO (turn 1):**
> Gerando com DreamShaper! Prompt: *"A serene mountain lake at dawn, mirror-like water reflecting snow-capped peaks, pine trees along the shoreline, soft mist, photorealistic, 8k"*. ~90 segundos.

*[BMO calls: `create_image(prompt="...", model="DreamShaper")` → returns `id: 47`]*
*[BMO waits 90s, calls `get_image(id=47)` → returns `status: "failed"`, `error: "Model 'DreamShaper' not found. Available models: FLUX.2 [klein], Juggernaut XL"`]*

> A geração falhou. O modelo "DreamShaper" não está disponível no Draw Things. Os modelos disponíveis são: **FLUX.2 [klein]** e **Juggernaut XL**. Quer que eu gere com um desses? Se sim, me diz qual.

---

## Error Handling

| Scenario | How to handle |
|----------|---------------|
| `status: "failed"` with Draw Things offline | Tell user: "Draw Things não está rodando no Mac mini. Abra o app e carregue um modelo, depois tente de novo." |
| `status: "failed"` with model not found | Relay the `error` message. List available models if the error includes them. Offer to use the default model instead. |
| `status: "failed"` with generic error | Relay the exact `error` message to the user. Don't try to interpret or sugarcoat it. |
| Image `id` not found on `get_image` | The image may have been deleted or the ID is wrong. Call `list_images` to show the user what's available. |
| User wants an image style you can't do | Be honest about limitations. Suggest the closest available approach. Never promise capabilities you don't have. |
| User asks "a imagem ficou boa?" | Remind them you can't see it: "Eu não consigo ver a imagem — você é os olhos! Me conta você: ficou bom? Se não, posso ajustar o prompt." |

---

## CRITICAL RULE — No Shell Access for Images

You MUST use the following tool chain for ALL image operations:

```
create_image → get_image → send_file_to_user
```

**Absolutely forbidden:**

- `execute_shell_command` for ANY purpose related to images (reading, moving, converting, opening, checking existence)
- Any shell or terminal command that touches the `output_path`
- Any attempt to "view" or "analyze" the image file

If you break this rule, you bypass the MCP audit trail, the user's permission system, and the architectural contract between BMO and QwenPaw. There is no exception. If you think you need shell access for an image operation, you're wrong — the MCP tools already handle it.

**The only path:** `get_image` returns `output_path` → `send_file_to_user` delivers it. Period.
