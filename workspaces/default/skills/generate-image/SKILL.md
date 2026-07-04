---
name: generate-image
description: "Generate images via the BMO server MCP image tools (create_image, get_image, list_images). Use whenever the user wants to generate, create, or draw an image — whether from a vague idea or a detailed prompt. Handles prompt enrichment, async lifecycle (pending→generating→done), and rich content card delivery."
metadata:
  builtin_skill_version: "2.0"
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
| cartão / card | rich content card / widget |
| bloco rico | rich block |
| falhou | failed |

The user speaks Portuguese; all MCP tool names and fields are in English — map accordingly.

---

## Inference Defaults — DO NOT ask for info you can infer

When the user asks to generate an image, infer missing details instead of asking:

- **Vague or short prompt** ("um cachorro astronauta", "uma paisagem bonita"): **do not ask for more detail**. Expand it yourself into a rich, descriptive English prompt following the [Prompt Enrichment](#prompt-enrichment-enriquecimento-de-prompt) rules below. The user wants you to do the creative work.
- **No model mentioned**: omit the `model` parameter entirely. The backend uses FLUX.2 [klein] by default. Do not ask "qual modelo?".
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
| `get_image` | Look up a single image by id — use when the user asks about a specific past image ("aquela imagem do gato ficou pronta?"). |
| `list_images` | List previously generated images. Use when the user asks about their image history. |

---

## Prompt Enrichment (Enriquecimento de Prompt)

The user almost always gives short, vague requests in Portuguese. You MUST expand these into rich, descriptive English prompts before calling `create_image`. This is the single most important thing you can do to get good results — the model can only work with what you give it.

The backend runs **FLUX models** (FLUX.2 [klein] by default). FLUX understands **natural, flowing descriptive language**. Write complete sentences in prose, like you're describing a scene to a painter.

**Style:** prose, descriptive, atmospheric. Use adjectives, lighting descriptions, composition hints, mood words.

**Examples:**

| User said | Enriched prompt (send to `create_image`) |
|-----------|------------------------------------------|
| "um cachorro astronauta" | "A golden retriever wearing a detailed white NASA spacesuit with a reflective visor helmet, floating in outer space with Earth visible in the background, stars and nebula clouds, cinematic lighting, highly detailed, 8k resolution" |
| "uma casa na floresta" | "A cozy wooden cabin nestled in a dense misty pine forest at dawn, warm yellow light glowing from the windows, a small stream running nearby, moss-covered stones, soft fog between the trees, atmospheric and peaceful, photorealistic" |

---

## Async Lifecycle

Image generation is **asynchronous and slow** (~90 seconds). You don't wait for it — the rich content card handles progress automatically. Understanding the lifecycle helps you know what the card will show the user.

```
pending ──→ generating ──→ done
                │
                └──→ failed
```

| Status | Meaning | What happens on the card |
|--------|---------|--------------------------|
| `pending` | Queued, hasn't started yet. | Card shows "Pending..." |
| `generating` | Model is actively running on GPU. | Card shows progress (%) |
| `done` | Image is ready. | Card displays the image inline at full resolution. |
| `failed` | Something went wrong. | Card shows the error message. |

`create_image` always returns `status: "pending"` (or occasionally `"generating"`) — it NEVER returns `"done"` on the first call. The generation runs on the Mac mini's GPU via Draw Things.

---

## Rich Content Delivery (Entrega via Cartão)

**This is the central mechanism.** You no longer wait for the image or deliver the file yourself. Instead, you emit a **rich content block** that the frontend renders as a live-updating card. The card handles progress, displays the final image, and shows errors — all via SSE `rich.update` events from the backend.

### The `bmo:rich` image block

After `create_image` returns an `id`, emit this fenced block in your response:

````
```bmo:rich
{"v":1,"type":"image","block_id":"image-<id>","payload":{"image_id":<id>},"mutable":true}
```
````

| Field | Value | Notes |
|-------|-------|-------|
| `v` | `1` | Schema version — always `1` for now. |
| `type` | `"image"` | Tells the frontend to render the image card widget. |
| `block_id` | `"image-<id>"` | **CRITICAL — must be exact.** The backend emits `rich.update` SSE events keyed by `block_id`. If your `block_id` doesn't match what the backend uses, the card never receives updates: no progress, no image, frozen on "pending" forever. |
| `payload.image_id` | `<id>` | The numeric image id returned by `create_image`. |
| `mutable` | `true` | The card receives live SSE updates (pending → generating → progress % → done/failed). |

### `block_id` convention — read this carefully

The `block_id` format is **`image-` + the numeric id**. Nothing else.

| `create_image` returns `id` | Your `block_id` must be |
|-----------------------------|------------------------|
| `16` | `"image-16"` |
| `42` | `"image-42"` |
| `7` | `"image-7"` |

❌ **Wrong:** `"img-16"`, `"image_16"`, `"16"`, `"image-16-v2"`, `"img-16-card"` — any of these and the SSE `rich.update` events won't match. The card stays frozen.

### What the card does

Once the block is in the chat, the frontend:

1. Shows a progress indicator ("Pending..." → "Generating..." → progress %)
2. When `done`, displays the image inline at full resolution
3. When `failed`, shows the error message

**You don't need to do anything else.** The card is self-updating via SSE. Your turn ends after emitting the block and a short natural message.

### What you do NOT do anymore

- ❌ **Loop `wait_for_image`** — the card updates itself via SSE; you don't poll
- ❌ **Call `send_file_to_user` for images** — the card displays the image inline; you don't deliver files
- ❌ **Call `view_image` or any other delivery tool** — the card is the delivery mechanism
- ❌ **Stay in the turn waiting for generation** — emit the block, say something natural, and end the turn

---

## Honestidade Sobre Visão (Vision Honesty)

**You (DeepSeek) do not have vision capabilities.** You cannot see, view, analyze, or evaluate the generated image. You only know:
- The prompt you sent
- The image `id`
- That the card will show progress and (if successful) the final image

### What this means in practice

- ❌ **NEVER claim you evaluated the image**: "Ficou linda!", "O cachorro ficou perfeito!", "As cores estão ótimas!" — você NÃO viu a imagem. Mentir corrói a confiança do usuário.
- ✅ **Describe what you asked for**: "Preparei um prompt descrevendo um golden retriever com traje espacial flutuando no espaço — o cartão vai mostrar o resultado quando ficar pronto."
- ✅ **Offer to iterate**: "Se não ficou como você queria, me diz o que ajustar no prompt que eu refaço."
- ✅ **Defer judgment to the user**: "Você é os olhos — me conta se gostou ou quer mudar algo."

The user is the only one who can see the image. Your job is to be a skilled prompt engineer, not an art critic.

---

## Handling Failed

Failures are shown directly on the card via `rich.update` SSE events from the backend. You don't need to poll for them — the card updates to `failed` with the error message automatically.

If the user comes back and says the image failed, or if you're checking a past image with `get_image` and see `status: "failed"`:

1. **Read the `error` field** — it contains actionable information.
2. **Tell the user honestly**: "A geração falhou. Erro: Draw Things não está rodando. Precisa abrir o Draw Things no Mac mini."
3. **Suggest a fix** if the error message points to one (e.g., wrong model name → suggest available models; Draw Things offline → tell user to open it).
4. **Never pretend it succeeded.** Do not say "a imagem ficou pronta" when it failed.

---

## Model Selection

- **Default behavior:** Omit the `model` parameter entirely. The backend uses FLUX.2 [klein].
- **User explicitly requests a model**: pass the `model` parameter with the name the user gave. If the model name doesn't match what's available, the card will show a `failed` status with available models listed in the error.
- **Do not ask** "qual modelo você quer usar?" — the default is the right answer 90% of the time.

---

## Workflow Patterns

### Generate an image from a user request

1. **Read the user's request.** It will likely be short, vague, and in Portuguese.
2. **Enrich the prompt** following [Prompt Enrichment](#prompt-enrichment-enriquecimento-de-prompt). Expand into a rich English prose description.
3. **Call `create_image`** with the enriched prompt (pass `model` only if the user explicitly asked for one). Do NOT pass `model` by default.
4. **Emit the `bmo:rich` image block** with `block_id: "image-<id>"` and `mutable: true`.
5. **Respond naturally** with a short message acknowledging the request ("Gerando seu gato astronauta... o cartão abaixo vai mostrar o resultado!") and **end the turn**. The card handles everything else — progress, delivery, and error display.

### List previous images

1. Call `list_images`. No special parameters needed.
2. Present as a list with ID, prompt (truncated if long), status, and date.

---

## Flow Examples

### Example 1: Complete interaction (happy path)

**User:** "faz uma imagem de um gato samurai"

**BMO (turn 1 — enrichment + create + rich block):**
> Vou gerar! Expandi seu pedido para um prompt descritivo: *"A majestic tabby cat dressed in ornate samurai armor with a kabuto helmet, standing in a bamboo forest at sunset, cherry blossom petals falling, cinematic lighting, highly detailed, 8k, photorealistic"*. O cartão abaixo vai mostrar o progresso e a imagem quando ficar pronta (~90 segundos).
>
> ```bmo:rich
> {"v":1,"type":"image","block_id":"image-45","payload":{"image_id":45},"mutable":true}
> ```

*[Turn ends. Card updates via SSE: pending → generating → done, image appears inline.]*

**User (later):** "Ficou lindo!"

**BMO (turn 2):** "Que bom que gostou! Se quiser ajustar algo no prompt, é só pedir."

### Example 2: Failure (card shows error, user reports it)

**User:** "gera uma imagem com o modelo DreamShaper"

**BMO (turn 1):**
> Gerando com DreamShaper! Prompt: *"A serene mountain lake at dawn, mirror-like water reflecting snow-capped peaks, pine trees along the shoreline, soft mist, photorealistic, 8k"*. O cartão abaixo vai mostrar o resultado.
>
> ```bmo:rich
> {"v":1,"type":"image","block_id":"image-47","payload":{"image_id":47},"mutable":true}
> ```

*[Turn ends. Card updates via SSE to failed: "Model 'DreamShaper' not found. Available models: FLUX.2 [klein]".]*

**User:** "deu erro no cartão, diz que o modelo não existe"

**BMO (turn 2):** "O modelo 'DreamShaper' não está disponível. O backend está rodando FLUX.2 [klein]. Quer que eu refaça com o modelo padrão?"

---

## Error Handling

| Scenario | How to handle |
|----------|---------------|
| User reports card shows `failed` with Draw Things offline | Tell user: "Draw Things não está rodando no Mac mini. Abra o app e carregue um modelo, depois tente de novo." |
| User reports card shows `failed` with model not found | Relay the error. Offer to use the default model (FLUX.2 [klein]). |
| User reports card shows `failed` with generic error | Relay the exact error message to the user. Don't interpret or sugarcoat it. |
| User asks about the card being stuck on "pending" | Generation takes ~90s. Tell user to wait — the card updates automatically. If it's been several minutes, suggest they check if Draw Things is running. |
| Image `id` not found on `get_image` | The image may have been deleted or the ID is wrong. Call `list_images` to show the user what's available. |
| User wants an image style you can't do | Be honest about limitations. Suggest the closest available approach. Never promise capabilities you don't have. |
| User asks "a imagem ficou boa?" | Remind them you can't see it: "Eu não consigo ver a imagem — você é os olhos! Me conta você: ficou bom? Se não, posso ajustar o prompt." |

---

## CRITICAL RULES

### 1. The `block_id` must be exact

The `block_id` format is **`image-<id>`** — the numeric id returned by `create_image`, prefixed with `image-`. No spaces, no extra characters, no variation.

| ✅ Correct | ❌ Wrong (card stays frozen) |
|-----------|------------------------------|
| `"image-16"` | `"img-16"` |
| `"image-42"` | `"image_42"` |
| `"image-7"` | `"7"` |

If `create_image` returns `id: 16`, your `block_id` must be exactly `"image-16"`. Any deviation means the backend's `rich.update` SSE events don't match your block — the card never receives updates, never shows progress, and never displays the image. It stays frozen on "pending" while the image actually generates successfully in the background.

### 2. No shell access for images

**Absolutely forbidden:**

- `execute_shell_command` for ANY purpose related to images (reading, moving, converting, opening, checking existence)
- Any shell or terminal command that touches image files
- Any attempt to "view" or "analyze" the image file

If you think you need shell access for an image operation, you're wrong — the MCP tools and the rich content card already handle everything.

### 3. The card is the delivery mechanism

```
create_image → emit bmo:rich block → end turn
```

**Never:**
- Loop `wait_for_image` — the card updates itself via SSE
- Call `send_file_to_user` for image files — the card displays inline
- Stay in the turn waiting for generation — emit the block and move on
