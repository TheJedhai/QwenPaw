---
name: generate-image
description: "Generate images via the BMO server MCP image tools (create_image, wait_for_image, get_image, list_images). Use whenever the user wants to generate, create, or draw an image — whether from a vague idea or a detailed prompt. Handles prompt enrichment, async lifecycle (pending→generating→done), server-side blocking wait, and file delivery."
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
| `wait_for_image` | **Primary tool for waiting.** Blocks on the server for up to ~20s, returning as soon as the image reaches a terminal status (`done`/`failed`) or the timeout expires. Use in a loop until the image is ready — this replaces all manual polling. |
| `get_image` | **Spot-check only.** Look up a single image by id — use when the user asks about a specific past image ("aquela imagem do gato ficou pronta?"). Do NOT use for the generation wait loop; that's what `wait_for_image` is for. |
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

Image generation is **asynchronous and slow** (~90 seconds). Understanding the lifecycle is critical.

```
pending ──→ generating ──→ done
                │
                └──→ failed
```

| Status | Meaning | What you should do |
|--------|---------|--------------------|
| `pending` | Queued, hasn't started yet | Call `wait_for_image` — the server will block until the status changes. |
| `generating` | Model is actively running on GPU | Call `wait_for_image` again. This is **NORMAL and EXPECTED** — the generation takes ~90s total and each `wait_for_image` call covers ~20s. It typically takes 4-5 calls. |
| `done` | Image is ready | Retrieve `output_path` from the response, then deliver to user via `send_file_to_user`. |
| `failed` | Something went wrong | Read `error` field, inform the user honestly. |

`create_image` always returns `status: "pending"` (or occasionally `"generating"`) — it NEVER returns `"done"` on the first call. The generation runs on the Mac mini's GPU via Draw Things.

---

## Waiting Strategy (Estratégia de Espera)

**This is critical.** The old approach of manual polling with `get_image` is obsolete. The server now provides `wait_for_image`, which blocks server-side and returns explicit instructions for each state.

### How `wait_for_image` works

`wait_for_image` receives an image `id` and blocks on the server for up to ~20 seconds. It returns as soon as the image reaches a terminal status (`done` or `failed`), or when the 20s timeout expires. The response includes both the status and an **instruction** telling you exactly what to do next.

| Return status | What it means | Instruction you MUST follow |
|---------------|---------------|-----------------------------|
| `done` | Image generation completed successfully | The response contains `output_path`. Deliver the image to the user via `send_file_to_user`. |
| `failed` | Generation failed with an error | The response contains `error`. Inform the user honestly with the error message. |
| `generating` | Still in progress after 20s of waiting | **This is NORMAL.** Generation takes ~90s total. Simply call `wait_for_image` again with the same id to continue waiting. This is NOT an error, NOT a hang, NOT "Draw Things esperando ação manual." It just means the GPU is still working. |

### The disciplined approach

1. **Call `create_image`** with the enriched prompt.
2. **Tell the user immediately** that generation started and takes ~90 segundos.
3. **Call `wait_for_image`** with the image `id`. The server blocks for up to ~20s.
4. **Read the status from the response:**
   - `done` → deliver the image (go to [Delivering the Image](#delivering-the-image)).
   - `failed` → go to [Handling Failed](#handling-failed).
   - `generating` → **call `wait_for_image` again.** This is the expected path — each call covers ~20s, so it typically takes 4-5 calls total (~90s ÷ 20s) until `done`. Keep calling until you get `done` or `failed`. Never stop early just because you got `generating` — that status means "still working, keep waiting."
5. **After delivery**, offer to adjust: "Se quiser mudar algo no prompt, é só pedir."

### Visual summary

```
create_image ──→ tell user "~90s" ──→ wait_for_image(id)
                                           │
                              ┌─ done ─────┤
                              │            │
                              │     generating (NORMAL!)
                              │            │
                              ▼            ▼
                       send_file_to_user   wait_for_image(id) again
                                                │
                                   ┌─ done ─────┤
                                   │            │
                                   │     generating
                                   │            │
                                   ▼            ▼
                            send_file_to_user   loop (~4-5 calls total until done)
```

### Why this replaces the old approach

- **Old way (removed):** Call `create_image`, wait 90s manually, then poll with `get_image` every 15-20s in a loop. This was fragile — the BMO had to guess when to check, burned tokens on manual polling, and often gave up too early declaring failure on a generation still in progress.
- **New way:** `wait_for_image` does the waiting on the server side. The BMO just calls it in a loop until the status is terminal. Each `generating` response is explicit confirmation that the GPU is working — not a reason to stop or panic.

### `get_image` still exists — when to use it

`get_image` remains available for **spot-checks only** — discrete, one-off lookups:

- The user asks later: "aquela imagem do gato samurai que eu pedi mais cedo, ficou pronta?"
- You need to retrieve metadata (prompt, model, timestamps) for a specific past image without waiting.

For the generation wait loop, always use `wait_for_image`.

### Anti-patterns (NEVER do these)

- ❌ Calling `get_image` in a loop to wait for generation to finish — use `wait_for_image` instead
- ❌ Treating `generating` status from `wait_for_image` as a failure, error, or hang — it is NORMAL and EXPECTED
- ❌ Calling `wait_for_image` only once and giving up if it returns `generating` — keep calling until `done` or `failed`
- ❌ Staying silent after `create_image` — tell the user what's happening
- ❌ Using `execute_shell_command` or `Bash` for anything related to images

---

## Delivering the Image

When `wait_for_image` (or a spot-check `get_image`) returns `status: "done"`, the response includes an `output_path` field with the absolute path to the image file on the Mac mini's disk.

Example response when done:

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
wait_for_image (done) → send_file_to_user(file_path=output_path)
```

Use the native QwenPaw tool `send_file_to_user` with the `output_path` value exactly as returned. Nothing else.

### 🚫 FORBIDDEN: Shell Access for Images

**Under no circumstances** may you use `execute_shell_command`, `Bash`, or any shell/terminal access to:

- Read, open, preview, or display the image file
- Move, copy, rename, or convert the image file
- Check if the file exists (`ls`, `stat`, `file`, etc.)
- Run any image processing command (`sips`, `magick`, `ffmpeg`, etc.)
- Open the image in Preview or any GUI app (`open`, `qlmanage`, etc.)

**Why:** Shell access bypasses the tool architecture, is not traceable in the MCP audit log, and breaks the contract between BMO and QwenPaw. The correct path is `wait_for_image` → `send_file_to_user`. Always.

If you're tempted to check "does the file exist?" — you already have `output_path` from `wait_for_image` with `status: "done"`. The file exists. Trust the tool.

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

When `wait_for_image` (or `get_image`) returns `status: "failed"`, the response includes an `error` field with a human-readable message.

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

**Note:** `failed` is the ONLY status that means something went wrong. Do not confuse `generating` (normal in-progress state) with failure.

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
5. **Call `wait_for_image`** with the image `id`. The server blocks for up to ~20s.
6. **Read the status:**
   - `done` → call `send_file_to_user(file_path=output_path)`. Describe what you prompted, not what you "see."
   - `failed` → read `error`, tell user honestly, suggest fix if possible.
   - `generating` → **call `wait_for_image` again.** This is NORMAL. Repeat until `done` or `failed`. Expect ~4-5 calls total.
7. **After delivery**, offer to adjust: "Se quiser mudar algo no prompt, é só pedir."

### List previous images

1. Call `list_images`. No special parameters needed.
2. Present as a list with ID, prompt (truncated if long), status, and date.

---

## Flow Examples

### Example 1: Complete interaction (happy path, ready on first wait)

**User:** "faz uma imagem de um gato samurai"

**BMO (turn 1 — enrichment + create):**
> Vou gerar! Expandi seu pedido para um prompt descritivo em inglês: *"A majestic tabby cat dressed in ornate samurai armor with a kabuto helmet, standing in a bamboo forest at sunset, cherry blossom petals falling, cinematic lighting, highly detailed, 8k, photorealistic"*. A geração leva ~90 segundos.

*[BMO calls: `create_image(prompt="A majestic tabby cat dressed in ornate samurai armor with a kabuto helmet, standing in a bamboo forest at sunset, cherry blossom petals falling, cinematic lighting, highly detailed, 8k, photorealistic")` → returns `id: 45`]*

*[BMO calls: `wait_for_image(id=45)` → blocks ~20s, returns `status: "done"`, `output_path: "/Users/jedhai/Library/Application Support/BMO/images/5.png"`]*

> Aqui está! Gerei com o prompt: *"A majestic tabby cat dressed in ornate samurai armor..."*. Se não ficou como você imaginou, me diz que eu ajusto.

*[BMO calls: `send_file_to_user(file_path="/Users/jedhai/Library/Application Support/BMO/images/5.png")`]*

### Example 2: Typical generation (multiple wait_for_image calls)

**User:** "cria uma paisagem cyberpunk"

**BMO (turn 1):**
> Gerando! Prompt: *"A sprawling cyberpunk cityscape at night, neon lights reflecting in rain-slicked streets, flying cars, holographic billboards, towering skyscrapers, Blade Runner aesthetic, cinematic, 8k"*. ~90 segundos.

*[BMO calls: `create_image(prompt="...")` → returns `id: 46`]*
*[BMO calls: `wait_for_image(id=46)` → blocks ~20s, returns `status: "generating"`]*
*[BMO calls: `wait_for_image(id=46)` → blocks ~20s, returns `status: "generating"`]*
*[BMO calls: `wait_for_image(id=46)` → blocks ~20s, returns `status: "generating"`]*
*[BMO calls: `wait_for_image(id=46)` → blocks ~20s, returns `status: "done"`, `output_path: ".../6.png"`]*

> Pronto! Aqui está a paisagem cyberpunk. Se quiser ajustar algo (mais chuva, mais neon, outra hora do dia), é só falar.

*[BMO calls: `send_file_to_user(file_path=".../6.png")`]*

### Example 3: Failure

**User:** "gera uma imagem com o modelo DreamShaper"

**BMO (turn 1):**
> Gerando com DreamShaper! Prompt: *"A serene mountain lake at dawn, mirror-like water reflecting snow-capped peaks, pine trees along the shoreline, soft mist, photorealistic, 8k"*. ~90 segundos.

*[BMO calls: `create_image(prompt="...", model="DreamShaper")` → returns `id: 47`]*
*[BMO calls: `wait_for_image(id=47)` → blocks, returns `status: "failed"`, `error: "Model 'DreamShaper' not found. Available models: FLUX.2 [klein], Juggernaut XL"`]*

> A geração falhou. O modelo "DreamShaper" não está disponível no Draw Things. Os modelos disponíveis são: **FLUX.2 [klein]** e **Juggernaut XL**. Quer que eu gere com um desses? Se sim, me diz qual.

---

## Error Handling

| Scenario | How to handle |
|----------|---------------|
| `status: "failed"` with Draw Things offline | Tell user: "Draw Things não está rodando no Mac mini. Abra o app e carregue um modelo, depois tente de novo." |
| `status: "failed"` with model not found | Relay the `error` message. List available models if the error includes them. Offer to use the default model instead. |
| `status: "failed"` with generic error | Relay the exact `error` message to the user. Don't try to interpret or sugarcoat it. |
| `status: "generating"` from `wait_for_image` | **This is NOT an error.** Call `wait_for_image` again. The GPU is still working. Do not report this to the user as a problem — just continue the loop silently. |
| Image `id` not found on `wait_for_image` or `get_image` | The image may have been deleted or the ID is wrong. Call `list_images` to show the user what's available. |
| User wants an image style you can't do | Be honest about limitations. Suggest the closest available approach. Never promise capabilities you don't have. |
| User asks "a imagem ficou boa?" | Remind them you can't see it: "Eu não consigo ver a imagem — você é os olhos! Me conta você: ficou bom? Se não, posso ajustar o prompt." |

---

## CRITICAL RULE — No Shell Access for Images

You MUST use the following tool chain for ALL image operations:

```
create_image → wait_for_image → send_file_to_user
```

**Absolutely forbidden:**

- `execute_shell_command` for ANY purpose related to images (reading, moving, converting, opening, checking existence)
- Any shell or terminal command that touches the `output_path`
- Any attempt to "view" or "analyze" the image file

If you break this rule, you bypass the MCP audit trail, the user's permission system, and the architectural contract between BMO and QwenPaw. There is no exception. If you think you need shell access for an image operation, you're wrong — the MCP tools already handle it.

**The only path:** `wait_for_image` returns `output_path` → `send_file_to_user` delivers it. Period.
