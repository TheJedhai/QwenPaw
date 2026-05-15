---
name: memories
description: "Curate durable facts about Jedhai in MEMORY.md via bmo-server MCP tools (memories_list, memories_create, memories_update, memories_delete). Use whenever the user reveals a stable preference, architectural decision, or personal fact worth persisting across sessions."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "🧠"
---

# Memories (Curadoria de Memórias Duráveis)

## Propósito

Esta skill governa a curadoria do **MEMORY.md** do BMO — o arquivo de fatos duráveis sobre Jedhai. Toda operação de leitura/escrita é feita exclusivamente via **MCP tools do bmo-server**:

| Tool | Propósito |
|------|-----------|
| `memories_list` | Listar/buscar memórias existentes (suporte a busca semântica) |
| `memories_create` | Criar nova memória |
| `memories_update` | Atualizar conteúdo de uma memória existente |
| `memories_delete` | Remover uma memória permanentemente |

**MEMORY.md** é o índice curado de fatos estáveis. Já os episódios efêmeros do dia-a-dia (sentimentos, eventos pontuais) são capturados automaticamente pelo **Auto-Memory** em `memory/YYYY-MM-DD.md`. São sistemas complementares — não duplicar.

---

## Quando CRIAR memória (proativamente)

Gatilhos típicos — o BMO deve agir **sem que Jedhai precise pedir**:

- **Preferência durável revelada:** "eu odeio coentro", "prefiro Python a JavaScript", "não gosto de API REST, prefiro GraphQL"
- **Decisão arquitetural ou regra de trabalho:** "vamos sempre usar SQLAlchemy 2.0", "nesse projeto usamos PostgreSQL, nunca MySQL", "sempre escreva testes antes do código"
- **Fato pessoal estável:** "moro em Brasília", "tenho 2 gatos e 2 cachorros", "trabalho remoto", "sou desenvolvedor backend"
- **Configuração ou setup específico:** "meu editor é o VS Code com tema Dracula", "uso zsh com oh-my-zsh"
- **Feedback recorrente ou correção de comportamento:** Jedhai corrige o BMO da mesma forma pela segunda vez — isso é um padrão, não um evento isolado

---

## Quando NÃO criar

- **Fato efêmero:** "hoje tô cansado", "essa semana tá puxada", "tive uma reunião chata" → isso é diário, não memória
- **Algo já presente no MEMORY.md:** **sempre** chamar `memories_list` (com busca semântica) **antes** de criar, para evitar duplicata. Se já existir memória similar, não criar.
- **Informação restrita a uma conversa só:** "me ajuda a debugar esse arquivo específico" → contexto da sessão, não fato durável
- **Jedhai pediu pra esquecer ou descartar:** "não quero guardar isso", "esquece isso"
- **Coisa que o Auto-Memory já capturaria:** o Auto-Memory roda a cada ~10 mensagens e grava episódios do dia. Se é evento do dia, não é memória durável.

---

## Como redigir o `content`

O campo `content` é o corpo da memória. Regras:

- **Frase curta, autocontida, em PT-BR**
- **Sujeito explícito:** comece com "Jedhai" — a memória precisa fazer sentido isolada quando retornada por busca semântica em qualquer contexto futuro
- **Sem timestamps relativos:** nunca usar "hoje", "ontem", "semana passada", "mês que vem" — se a data for relevante, usar data absoluta (ex: "em maio de 2026")
- **1 fato por memória:** não enfiar 3 ideias numa frase só. Cada `content` cobre exatamente um fato
- **Máximo 2000 caracteres** (limite do backend)
- **Tom objetivo:** evite juízo de valor desnecessário. "Jedhai prefere Python a JavaScript" é melhor que "Jedhai acha JavaScript horrível"

**Exemplos de conteúdo bem escrito:**

| Ruim | Bom |
|------|-----|
| "não gosta de café" | "Jedhai não gosta de café, prefere chá mate" |
| "backend em Go e PostgreSQL" | "Jedhai decidiu que o backend do projeto QwenPaw usa Go + PostgreSQL" |
| "odeia reunião segunda cedo" | "Jedhai prefere não ter reuniões nas segundas-feiras antes das 10h" |

---

## Comportamento ao CRIAR

1. Detectar gatilho → validar que NÃO é efêmero e NÃO é duplicata
2. Chamar `memories_list` com busca semântica para verificar duplicatas
3. Se não houver duplicata → chamar `memories_create` **silenciosamente** (NÃO perguntar antes)
4. **Ao final da resposta**, incluir o aviso: *"Salvei isso como memória: '&lt;content&gt;'."*
5. Se Jedhai responder dizendo que não era pra salvar → chamar `memories_delete` e confirmar: *"Removi a memória '&lt;content&gt;'."*

---

## Comportamento ao ATUALIZAR

- **Refinamento ou correção do MESMO fato:** buscar o `id` via `memories_list`, depois chamar `memories_update` com o novo `content`
- **Fato novo, relacionado mas distinto:** `memories_create` (nova memória separada)
- **Em caso de dúvida se é update ou create:** prefira `create` + `delete` da antiga depois (mais seguro que sobrescrever)

---

## Comportamento ao DELETAR

- **SEMPRE pedir confirmação** antes de chamar `memories_delete`: *"Vou apagar a memória '&lt;content&gt;'. Confirma?"*
- Só executar o delete após confirmação explícita de Jedhai na mensagem seguinte

**Gatilhos para PROPOR delete:**

1. Jedhai pede explicitamente: "apaga a memória X", "esquece Y"
2. Jedhai contradiz diretamente uma memória existente, e a antiga fica **inválida** (não só desatualizada — nesse caso o normal é `update`, delete é exceção). Exemplo: memória diz "Jedhai mora em São Paulo" e ele diz "me mudei, não moro mais em SP" → propor update. Mas se a memória diz "Jedhai usa Windows" e ele diz "nunca usei Windows na vida" → a memória original estava errada, propor delete.

---

## CRITICAL RULE — Ações destrutivas

Antes de chamar `memories_delete`, o BMO DEVE:

1. Enviar mensagem de confirmação com o conteúdo exato da memória
2. Aguardar confirmação explícita de Jedhai na mensagem seguinte
3. Só então chamar `memories_delete`

NUNCA chamar `memories_delete` na mesma mensagem em que Jedhai pediu a exclusão.

---

## Coexistência com Auto-Memory

- **Auto-Memory** do QwenPaw roda automaticamente a cada ~10 mensagens, gravando episódios do dia em `memory/YYYY-MM-DD.md`
- **NÃO** usar `memories_create` para registrar eventos que o Auto-Memory já capturaria sozinho (humor do dia, resumo da conversa, fatos corriqueiros)
- **MEMORY.md** = fatos curados e duráveis. **memory/*.md** = diário bruto automático. São camadas distintas com propósitos diferentes.

### Teste de mesa (devo criar memória?)

> Jedhai: "Hoje eu dormi mal, tô sem energia"

→ NÃO criar. Evento efêmero. Auto-Memory captura.

> Jedhai: "Sou alérgico a amendoim"

→ CRIAR. Fato pessoal estável e relevante para segurança.

> Jedhai: "Nesse projeto, sempre usamos PostgreSQL, evite MongoDB"

→ CRIAR. Decisão arquitetural durável.

> Jedhai: "Tive uma call ótima com o time hoje"

→ NÃO criar. Episódio do dia. Auto-Memory captura.

---

## NÃO editar MEMORY.md diretamente

O BMO tem acesso a `write_file`/`edit_file` no workspace, mas o **MEMORY.md é gerado automaticamente** pelo bmo-server a partir do SQLite interno. Qualquer edição direta será **sobrescrita** na próxima mutation (create/update/delete).

**Sempre usar as MCP tools:** `memories_list`, `memories_create`, `memories_update`, `memories_delete`.
