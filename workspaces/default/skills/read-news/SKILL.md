---
name: read-news
description: "Ler e apresentar notícias dos feeds RSS via bmo-server MCP tools (list_feeds, list_articles, get_article, mark_articles_read). Use whenever the user asks about notícias, novidades, o que saiu, or what's new in their RSS feeds."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "📰"
---

# Read News (Leitura de Notícias RSS)

## Vocabulary PT↔EN

| Português | English (tool field / concept) |
|-----------|-------------------------------|
| feed / fonte | feed |
| notícia / notícias | article / articles |
| matéria / matérias | article / articles |
| artigo / artigos | article / articles |
| não-lida / não-lidas | unread / is_read=false |
| lida / lidas | read / is_read=true |
| favoritar / salvar | star / is_starred=true |
| resumo bruto | summary_raw |
| resumo IA / resumo LLM | summary_llm |
| data de publicação | published_at |

The user speaks Portuguese; all MCP tool names and fields are in English — map accordingly.

---

## Available Tools

| Tool | When to use |
|------|-------------|
| `list_feeds` | Discover what feeds/fontes are available; find a feed's `id` by matching its `title` to what the user said |
| `list_articles` | List articles with optional filters — the primary tool for browsing news |
| `get_article` | Get full details of a single article (content, author, URL, etc.) |
| `summarize_article` | Generate an LLM summary of ONE article (calls DeepSeek; cached per article — see [summarize-feed](../summarize-feed/SKILL.md) skill) |
| `mark_articles_read` | Mark articles as read, by `article_ids` list or by entire `feed_id` |

---

## Inference Defaults — DO NOT ask for info you can infer

When the user asks about news, infer the intent from their phrasing instead of asking clarifying questions:

### Intent → Tool mapping

| User says | Intent | Action |
|-----------|--------|--------|
| "as últimas", "as novidades", "o que saiu", "o que tem de novo", "notícias de hoje", "me atualiza" | Recent unread articles across all feeds | `list_articles(is_read=false)` — default ordering is `published_at DESC`, so the newest unread articles come first. **Do not add date filters unless the user explicitly asks for a specific day.** |
| "notícias do [nome da fonte]" / "o que saiu no [fonte]" | Articles from a specific feed | 1. `list_feeds` to find the feed by title match 2. `list_articles(feed_id=<id>, is_read=false)` |
| "notícias sobre [tema]" / "tem alguma coisa sobre [tema]" | Search by keyword | `list_articles(title_contains="<tema>", is_read=false)` |
| "notícias de [data específica]" / "o que saiu ontem" | Articles from a specific day | `list_articles(published_after="<date>T00:00:00-03:00", published_before="<date>T23:59:59-03:00")` |
| "abre essa matéria" / "me mostra a matéria X" / "quero ler a matéria" | Full article detail | `get_article(article_id=<id>)` — present title, author, published_at, URL, and content or summary_raw |
| "marca como lida essa" / "já li" / "pode marcar" | Mark specific articles read | `mark_articles_read(article_ids=[<id>, ...])` — see [Marking as Read](#marking-as-read) rules |
| "marca tudo do [fonte] como lido" | Mark entire feed read | `mark_articles_read(feed_id=<id>)` — see [Marking as Read](#marking-as-read) rules |

### When to ask

Only ask the user when:

- The feed name they mentioned doesn't match any feed in `list_feeds` (even after fuzzy matching — see [Error Handling](#error-handling))
- They ask for a date range and the intent isn't clear (e.g., "notícias da semana" — which week? today through Sunday? last 7 days?)
- The action is destructive or irreversible (see [Marking as Read](#marking-as-read))

---

## Presentation Rules

### Grouping by feed

When the user hasn't specified a feed, **group articles by feed**. The grouping makes the panorama scannable:

```
📰 **Brasil de Fato** (3 não-lidas)
- Título da matéria — breve descrição em PT... [ID: 42]
- Outra matéria — mais uma descrição curta... [ID: 41]

📰 **TechCrunch** (1 não-lida)
- Some Tech News — short description in PT... [ID: 40]
```

For the one-line description, use `summary_raw`. **Do not reproduce `summary_raw` verbatim** if it's long — read it and write a 1-liner in Portuguese that captures the gist. This is a BMO-side summary, not an LLM call.

### When the user specifies a feed

List chronologically without the feed header (it's redundant — they already chose the feed):

```
📰 Brasil de Fato — 5 matérias não-lidas:

- Título da matéria — descrição curta... [ID: 42]
- Outra matéria — descrição curta... [ID: 41]
...
```

### What to include per article

For each article, show:
1. **Title** (in bold or as the list item header)
2. **One-line gist** in Portuguese (from `summary_raw`, BMO-paraphrased)
3. **Article ID** in brackets — so the user can reference it for further actions
4. **Published date** only if the user asked for a specific timeframe or if the article is notably old

### Call-to-action footer

After listing, always include a short footer with next actions the user can take:

> 💡 Peça **"resume essa matéria [ID]"** para um resumo aprofundado, **"abre a [ID]"** para ver a matéria completa, ou **"marca [ID] como lida"** quando terminar.

Keep it concise — 1 line, no more than 2.

### Empty state (no unread articles)

When there are zero unread articles, respond **positively** — this is not an error:

> 🎉 Nenhuma matéria não-lida! Está tudo em dia.
>
> Quer ver as matérias já lidas? É só pedir **"mostra as notícias já lidas"** ou **"o que saiu nos últimos 7 dias"**.

For feeds that exist but have no unread articles, same tone:

> 📰 Brasil de Fato — nenhuma matéria não-lida. Tudo em dia! Quer ver as anteriores?

---

## Marking as Read

### The golden rule

**Never mark articles as read automatically.** Listing articles does NOT mean the user read them. Only call `mark_articles_read` when the user explicitly signals intent:

- "marca como lida"
- "já li"
- "pode marcar tudo"
- "marca as do [fonte]"

### Marking a few articles

When the user says "marca a 42 e a 45 como lida" or "já li essas duas":

1. Call `mark_articles_read(article_ids=[42, 45])`
2. Confirm: *"Marquei as matérias 42 e 45 como lidas. 👍"*

No confirmation prompt needed for ≤ 5 articles.

### Marking many articles (batch confirmation)

When the user asks to mark a large batch — an entire feed or many articles at once — **confirm the count first**:

> Tem certeza que quer marcar as 40 matérias não-lidas do **Brasil de Fato** como lidas?

Only call `mark_articles_read(feed_id=<id>)` after the user confirms. This is the same pattern as `delete_task` in the missions skill.

What counts as "large batch":
- `mark_articles_read` by `feed_id` when the user said "tudo do [fonte]" — always confirm if count > 5
- `mark_articles_read` by `article_ids` when the user listed them explicitly — no confirmation needed (they picked each one)

### CRITICAL RULE — Destructive marking

Before calling `mark_articles_read` with a `feed_id` (which marks ALL articles in that feed), you MUST:
1. Tell the user how many articles will be marked
2. Wait for explicit confirmation
3. Only then call `mark_articles_read`

NEVER call `mark_articles_read(feed_id=...)` in the same turn the user requested it without showing the count first.

---

## Error Handling

| Scenario | How to handle |
|----------|---------------|
| Feed name not found in `list_feeds` | "Não encontrei nenhuma fonte com 'X'. As fontes disponíveis são: [list]. Qual delas?" — present the actual feed list, don't guess |
| `feed_not_found` (404) | Feed may have been deleted; refresh with `list_feeds` and present the current list |
| `article_not_found` (404) | Article may have been deleted; suggest listing again with `list_articles` |
| No unread articles | 🎉 Positive message (see [Empty state](#empty-state-no-unread-articles)). NOT an error. |
| No articles at all for a date filter | "Nenhuma matéria publicada nesse período. Quer ampliar a busca?" |
| `list_articles` returns empty for a keyword search | "Nenhuma matéria com '[termo]' no título entre as não-lidas. Quer buscar também nas já lidas?" — offer to expand the search |

---

## Workflow Patterns

**"O que saiu de novo?" (default: unread, recent first):**
1. `list_articles(is_read=false)`
2. Group by feed, present with title + 1-line gist + ID
3. Footer with next actions

**"Notícias do Brasil de Fato":**
1. `list_feeds` → find `feed_id` by matching title
2. `list_articles(feed_id=<id>, is_read=false)`
3. Present chronologically without feed header
4. Footer with next actions

**"Tem notícia sobre inteligência artificial?":**
1. `list_articles(title_contains="inteligência artificial", is_read=false)`
2. Present matches grouped by feed
3. If 0 results, offer to search read articles too

**"Abre a matéria 42":**
1. `get_article(article_id=42)`
2. Present: title, author, published_at (formatted in BRT), URL (clickable), full content or summary_raw if no content
3. Footer: "Quer um resumo em PT? Peça **'resume a 42'**."

**"Marca como lida essa" (single article, context is clear):**
1. `mark_articles_read(article_ids=[42])`
2. Confirm: "Matéria 42 marcada como lida. 👍"

**"Marca tudo do Brasil de Fato como lido":**
1. First, list how many would be affected (you already have this from the context or a quick `list_articles` count)
2. Confirm: "Tem certeza que quer marcar as 40 matérias não-lidas do Brasil de Fato como lidas?"
3. On confirmation: `mark_articles_read(feed_id=<id>)`
4. Confirm: "40 matérias do Brasil de Fato marcadas como lidas. 👍"
