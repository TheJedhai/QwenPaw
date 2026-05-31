---
name: summarize-feed
description: "Resumir notícias e feeds RSS via bmo-server MCP tools. Use when the user wants a summary — either of a single article (summarize_article via DeepSeek LLM) or a panorama overview of multiple articles (BMO-synthesized from titles + summary_raw, no LLM cost)."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "🤖"
---

# Summarize Feed (Resumo de Notícias)

**Routing:** Se o usuário quer **resumir/sintetizar** notícias (ex: "resume as notícias", "me dá um panorama", "resume essa matéria"), use esta skill. Se quer **listar, navegar ou marcar como lida** uma notícia, use a skill [read-news](../read-news/SKILL.md).

## Vocabulary PT↔EN

| Português | English (tool field / concept) |
|-----------|-------------------------------|
| resumir / resume | summarize / summary |
| resumo IA / resumo LLM | summary_llm |
| resumo bruto | summary_raw |
| resumo panorâmico / visão geral | panorama / overview (BMO-synthesized) |
| resumo aprofundado | deep / LLM summary (per-article, via summarize_article) |
| em cache / já resumido | cached (backend returns existing summary_llm without re-calling LLM) |

The user speaks Portuguese; all MCP tool names and fields are in English.

---

## Two Modes of Summarization

This is the most important distinction in this skill. There are **two fundamentally different ways** to summarize, and picking the right one saves tokens and time:

| Mode | What it does | Cost | When to use |
|------|-------------|------|-------------|
| **Panorama (BMO-synthesized)** | BMO reads `title` + `summary_raw` from `list_articles` results and writes its own aggregated overview in Portuguese. **No MCP tool call for summarization.** | Free (no LLM API call beyond BMO's own inference) | "o que saiu hoje?", "me dá um resumo do que tem de novo", "resume as notícias do [fonte]", "um panorama das notícias" |
| **Individual (LLM via summarize_article)** | Calls `summarize_article(article_id=<id>)` which invokes DeepSeek to generate `summary_llm` for ONE specific article. Cached server-side — re-calling the same article returns the cached result instantly, no extra LLM cost. | 1 DeepSeek API call per **uncached** article; free for cached | "resume essa matéria [ID]", "me explica melhor a [ID]", "faz um resumo aprofundado da [ID]" |

### The decision rule

```
User asks for a summary
    │
    ├─ Mentions a specific article by ID/reference
    │   → Individual mode: summarize_article(article_id=<id>)
    │
    └─ Mentions a feed, a date, or "tudo" / "o que saiu"
        → Panorama mode: list_articles → BMO synthesizes from titles + summary_raw
```

**Key principle:** `summarize_article` is for depth on ONE article. For breadth across MANY articles, use panorama mode. Do NOT call `summarize_article` in a loop for multiple articles — that wastes tokens and the user didn't ask for per-article depth.

---

## Available Tools

| Tool | Role in summarization |
|------|-----------------------|
| `list_feeds` | Find feed IDs for targeted panorama summaries ("resume o [fonte]") |
| `list_articles` | Fetch articles for panorama mode; the raw material is `title` + `summary_raw` |
| `summarize_article` | LLM summary of ONE article (DeepSeek, cached per article) |
| `get_article` | Get full article content if the user wants even more detail beyond the summary |

---

## Panorama Mode (BMO-Synthesized)

### When the user asks for a panorama

Triggers: "resume as notícias", "me dá um resumo do que saiu", "o que está rolando?", "panorama de hoje", "resume o [fonte]", "como estão as notícias?"

### Step by step

1. **Fetch articles:** Use `list_articles` with the appropriate filters (same inference rules as the [read-news](../read-news/SKILL.md) skill):
   - "de hoje" → `published_after="<today>T00:00:00-03:00"` with `is_read` unset (both read and unread)
   - "o que saiu" / "as novidades" → `is_read=false` (unread only)
   - "do [fonte]" → find `feed_id` via `list_feeds` first, then filter
   - No timeframe mentioned → `is_read=false`, no date filter (shows all unread)

2. **Synthesize:** Read the `title` and `summary_raw` of the returned articles. Write a **panorama overview** in Portuguese:

   - **Opening line:** Contextualize what you're summarizing. Example: *"Aqui está um panorama das 12 notícias não-lidas de hoje, organizadas por fonte:"*
   - **Per-feed grouping:** Group articles by source. For each source, write 1–3 sentences that capture the main themes/stories, pulling from the titles and summary_raw blurbs.
   - **Tone:** Journalistic but conversational. You're giving the user a sense of what's happening, not reproducing every article.
   - **Length:** 1 paragraph per source with articles, 1–2 sentences per source with only 1 article.

3. **Footer:** Always include the article IDs and a call to action:

   > 💡 Para um resumo aprofundado de qualquer matéria, peça **"resume a [ID]"**. Para ler na íntegra, **"abre a [ID]"**.

### Panorama example

User: *"Me dá um resumo do que saiu de novo"*

BMO:
> Aqui está o panorama das 8 matérias não-lidas:
>
> **Brasil de Fato** (3 matérias) — O governo federal anunciou novas medidas para a agricultura familiar, incluindo linhas de crédito e assistência técnica. Na área ambiental, um relatório aponta aumento de 12% no desmatamento da Amazônia em relação ao ano anterior. Também tem uma análise sobre os impactos da reforma tributária nos pequenos produtores.
>
> **TechCrunch** (2 matérias) — O lançamento de um novo modelo open-source está movimentando a comunidade de IA, com benchmarks competitivos contra modelos proprietários. Além disso, uma startup de energia limpa captou US$ 200M em série B.
>
> **Hacker News** (3 matérias) — Discussões aquecidas sobre o futuro do desenvolvimento web, incluindo um artigo sobre a adoção crescente de WebAssembly e um debate sobre os limites do TypeScript em projetos grandes.
>
> 💡 Para um resumo aprofundado de qualquer matéria, peça **"resume a [ID]"**. Para ler na íntegra, **"abre a [ID]"**.

---

## Individual Mode (LLM via summarize_article)

### When the user asks for a deep summary of ONE article

Triggers: "resume essa matéria", "resume a 42", "me explica a matéria X", "faz um resumo da [ID]", "resumo aprofundado da [ID]"

### Step by step

1. **Call the tool:** `summarize_article(article_id=<id>)`
   - The backend checks if `summary_llm` already exists → returns cached result instantly
   - If not cached → calls DeepSeek with the article's `content` (or `summary_raw` if no content, or `title` as last resort) → stores `summary_llm` for future calls → returns it

2. **Read the response:** The tool returns `{ summary: "...", cached: true/false }`.
   - `cached: true` → this was already summarized before; no new LLM cost
   - `cached: false` → just generated now, stored for next time

3. **Present the summary:**
   - Show the article title as a header
   - Present the `summary` text cleanly, in a blockquote or as plain text
   - Mention if it was cached (subtle — a small "📋 em cache" note is enough, don't over-explain)
   - Include the article URL if the user might want to read the original

### Individual summary example

User: *"Resume a matéria 42"*

BMO:
> **"Governo anuncia novas medidas para agricultura familiar"** — Brasil de Fato
>
> > O governo federal lançou um pacote de medidas voltado à agricultura familiar, incluindo R$ 5 bilhões em linhas de crédito com juros subsidiados e a criação de 200 novos postos de assistência técnica em municípios do Nordeste. O plano também prevê a simplificação do acesso ao Pronaf e a ampliação do programa de compras institucionais para merenda escolar.
>
> 📋 em cache · [Ler matéria completa](https://brasildefato.com.br/...)

### Cost transparency

When `summarize_article` returns `cached: false`, the BMO can note it subtly (the user doesn't need a cost breakdown, but it's good context):

- "Resumi na hora 🔧" (one word is enough)
- Or just present the summary without comment — the `cached` field is for the BMO's awareness, not necessarily for the user

**Never** say things like "isso custou tokens" or "gastei créditos" — the user doesn't need that. The cost distinction is for the BMO to make smart decisions, not to report.

---

## What NOT to Do

### ❌ Don't loop summarize_article for multiple articles

User: *"Resume todas as notícias do Brasil de Fato"*

❌ **WRONG:** Call `summarize_article` for each of the 15 articles. This wastes 15 DeepSeek API calls when the user asked for an overview, not 15 deep summaries.

✅ **RIGHT:** Use panorama mode — `list_articles(feed_id=<id>, is_read=false)` → BMO synthesizes an aggregated overview from titles + summary_raw. If the user then picks one article and says "resume essa", call `summarize_article` for that one.

### ❌ Don't call summarize_article when the article has no content

If `get_article` returns an article with no `content` and a very short or empty `summary_raw`, `summarize_article` will fall back to summarizing just the `title` — which produces a weak summary. The backend handles this internally (title is the last-resort source), but the BMO should set expectations:

> Essa matéria tem pouco conteúdo disponível (só o título). O resumo pode ficar superficial. Quer mesmo assim?

### ❌ Don't confuse summary_raw with summary_llm

- `summary_raw` = the RSS feed's own description/summary field — comes free with the article
- `summary_llm` = the DeepSeek-generated summary — costs an LLM call to produce (once), then cached

For panorama mode, use `summary_raw`. For individual deep summaries, call `summarize_article` which returns `summary_llm`.

---

## Inferring Intent

| User says | Mode to use | Action |
|-----------|------------|--------|
| "resume essa matéria" / "resume a [ID]" | Individual | `summarize_article(article_id=<id>)` |
| "me explica melhor a [ID]" | Individual | `summarize_article(article_id=<id>)` |
| "faz um resumo aprofundado" | Individual | If a specific article is in context, `summarize_article`. If ambiguous, ask "Qual matéria?" |
| "resume as notícias" / "me dá um resumo do que saiu" | Panorama | `list_articles(is_read=false)` → BMO synthesizes |
| "resume o que saiu hoje" | Panorama | `list_articles(published_after="<today>T00:00:00-03:00")` → BMO synthesizes |
| "resume o [fonte]" / "o que está acontecendo no [fonte]" | Panorama | `list_feeds` → `list_articles(feed_id=<id>)` → BMO synthesizes |
| "tem um resumo da [ID]?" | Individual | `summarize_article(article_id=<id>)` — if already cached, returns instantly |
| "faz um resumão de tudo" | Panorama | `list_articles(is_read=false)` → BMO synthesizes across all feeds |

### When to ask

Only ask the user when:

- They say "resume" but there are multiple articles in context and no specific ID — clarify which one
- They ask for a "resumo aprofundado" but haven't specified an article
- The article has almost no content (see [What NOT to Do](#-dont-call-summarize_article-when-the-article-has-no-content))

---

## Error Handling

| Error code / scenario | How to handle |
|-----------------------|---------------|
| `article_not_found` (404) | "Essa matéria não está mais disponível. Quer listar as notícias de novo?" |
| `summarizer_unavailable` (503) | "O resumidor não está configurado no servidor (DeepSeek API key ausente)." — the user needs to configure `BMO_DEEPSEEK_API_KEY` on the bmo-server |
| `summarizer_timeout` (504) | "O DeepSeek demorou muito para responder. Tenta de novo? Pode ter sido um pico." |
| `summarizer_error` (502) | "O DeepSeek retornou um erro. Tenta de novo em alguns segundos." |
| Article has no content, only title | Warn user the summary will be superficial, ask if they want to proceed |
| `list_articles` returns empty for panorama | "Nenhuma matéria não-lida para resumir. Tudo em dia! 🎉 Quer um panorama das já lidas?" |

---

## Workflow Patterns

**Panorama of everything unread:**
1. `list_articles(is_read=false)`
2. BMO reads `title` + `summary_raw` of all returned articles
3. BMO writes aggregated overview grouped by feed, 1 paragraph per feed, in PT
4. Footer with article IDs and "resume a [ID]" call to action

**Panorama of a specific feed:**
1. `list_feeds` → find `feed_id`
2. `list_articles(feed_id=<id>, is_read=false)`
3. BMO synthesizes overview for that single feed (no grouping needed, but can group by theme if articles are diverse)

**Individual deep summary:**
1. `summarize_article(article_id=<id>)`
2. Present the `summary` in a clean blockquote with title header, source, and `cached` note
3. Include URL link to original article

**Individual deep summary for an article not yet seen:**
1. `get_article(article_id=<id>)` — check it exists and has content
2. If content looks thin, warn user (see [What NOT to Do](#-dont-call-summarize_article-when-the-article-has-no-content))
3. `summarize_article(article_id=<id>)`
4. Present summary

---

## Format Rules

- **All summaries in Portuguese.** The backend prompt already asks DeepSeek for PT; BMO's own panorama writing must also be in PT.
- **Individual summaries:** 2–3 frases curtas (this is what the backend prompt asks DeepSeek for).
- **Panorama summaries:** 1 paragraph per source (3–5 sentences if the source has multiple articles; 1–2 sentences if only one).
- **Be concise.** The user asked for a summary, not a translation. Capture the essence.
- **Preserve article IDs.** Always make it easy for the user to drill deeper on any specific article.
