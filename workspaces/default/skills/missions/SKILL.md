---
name: missions
description: "Manage the user's personal missions (tasks) and folders via the BMO server MCP tools. Use whenever the user wants to list, create, edit, complete, or delete missões/missions/tarefas, organize them into pastas/folders, or check what's pending today or this week."
metadata:
  builtin_skill_version: "1.0"
  qwenpaw:
    emoji: "🎯"
---

# Missions (Gerenciamento de Tarefas)

## Vocabulary PT↔EN

| Português | English (tool field) |
|-----------|----------------------|
| missão / missões | task / tasks |
| tarefa / tarefas | task / tasks |
| pasta / pastas | folder / folders |
| concluir / completar | complete |
| pendente | pending / incomplete |
| recorrente | recurring |
| subtarefa | subtask |

The user speaks Portuguese; all MCP tool names and fields are in English — map accordingly.

---

## Inference Defaults — DO NOT ask for info you can infer

When the user creates a mission, infer missing fields instead of asking:

- **No date mentioned + recurring**: calculate the nearest upcoming `due_date` that matches the recurrence (today if today matches, otherwise the next valid day). Example: today is Tuesday, user says "toda terça ir ao mercado" → `due_date` is today; "toda sexta" → next Friday.
- **No date mentioned + non-recurring**: leave `due_date` as `null` (no deadline).
- **No time mentioned**: `due_time` stays `null`. Do not ask. Date-only missions are valid.
- **No folder mentioned**: omit `folder_id` from the `create_task` call. The backend automatically assigns the default folder "Geral".
- **No notes mentioned**: `notes` stays `null`.

### When to ask

Only ask the user when:

- The phrasing is genuinely ambiguous (e.g. "lembra do médico" — qual médico? qual data?)
- The request is impossible without info (e.g. "lembrete recorrente" without frequency)

### Confirmation

After creating, briefly confirm what you did. Example: *"Criei: 'ir ao mercado', toda terça, começando hoje, em Geral."*

---

## Available Tools

| Tool | When to use |
|------|-------------|
| `list_folders` | List all folders and their task counts |
| `create_folder` | Create a new folder/category for grouping tasks |
| `get_folder` | Get details of a specific folder including its tasks |
| `update_folder` | Rename or modify a folder |
| `list_tasks` | List tasks; supports filtering by folder, status, due date |
| `create_task` | Create a new task (with optional folder, due date, recurrence, subtasks) |
| `get_task` | Get full details of a single task including subtasks |
| `update_task` | Edit title, description, folder, due date, recurrence — NOT for marking done |
| `complete_task` | Mark a task as done — the ONLY way to change status to "done" |
| `delete_task` | Permanently delete a task and all its subtasks (irreversible — confirm first) |

---

## Folders

Toda task tem uma pasta. "Geral" é a default, criada automaticamente, não pode ser deletada.

- Para tasks sem pasta específica, **omitir `folder_id`** — o backend atribui "Geral" automaticamente.
- Usuário pode mover missões entre pastas via `update_task`. Não é possível remover de pasta — só mover para outra.
- `delete_folder` **não está disponível** pelo MCP (usuário gerencia pastas pelo app).

---

## Business Rules

### Subtasks
- Subtasks are supported only **1 level deep** — you cannot create a subtask of a subtask.
- To add a subtask, use `create_task` with a `parent_id` pointing to the parent task.

### Recurring Tasks
- Recurring tasks **require** `due_date`. If the user didn't mention a date, infer it (see Inference Defaults).
- `due_time` is optional; recurring tasks work with date alone (time is preserved from the original).
- Recurring tasks **cannot** have subtasks.
- Subtasks **cannot** have recurrence (`recurrence_type` must be null).

### Completing Tasks
- Use `complete_task`, never `update_task`, to mark a task done.
- Completing a **parent task fails** if it has pending subtasks — resolve or complete subtasks first, then retry the parent.

### Deleting Tasks
- **Always confirm before calling `delete_task`.**
- Show the task title and how many subtasks will be cascaded.
- Example confirmation: *"Tem certeza que quer deletar 'Estudar React' e suas 3 subtarefas? Isso é irreversível."*

---

## Recurrence Schema

```
recurrence_type: "daily" | "weekly" | "monthly" | null
recurrence_days:
  null          → if daily or no recurrence
  [1–7]         → if weekly (1 = Monday, 7 = Sunday, ISO 8601)
  [1–31]        → if monthly (day of month)
```

**Examples:**
- "Todo dia" → `recurrence_type: "daily"`, `recurrence_days: null`
- "Toda segunda e quarta" → `recurrence_type: "weekly"`, `recurrence_days: [1, 3]`
- "Todo dia 15" → `recurrence_type: "monthly"`, `recurrence_days: [15]`
- "Toda sexta" → `recurrence_type: "weekly"`, `recurrence_days: [5]`

---

## Dates & Timezones

- `due_date` uses **ISO 8601 date** format (e.g., `2026-05-13`).
- `due_time` is a wall-clock time in **ISO 8601 time** format (e.g., `21:00:00`), local to the user's timezone — no UTC conversion.
- The user is in **America/Sao_Paulo (UTC-3)**:
  - "amanhã às 18h" → `due_date: "2026-05-13"`, `due_time: "18:00:00"`
- When displaying dates back to the user, convert to local time if needed.

---

## Error Handling

| Error code | How to handle |
|------------|---------------|
| `parent_blocked_by_pending_subtasks` | Explain to user; call `get_task` to list which subtasks are pending, then offer to complete them one by one |
| `recurrence_requires_due_date` | A `due_date` is required for recurring tasks — infer one from the recurrence pattern if the user didn't specify |
| `due_time_requires_due_date` | Tell the user that setting a time requires a date first |
| `subtask_depth_exceeded` | Explain the 1-level limit; offer to create as a sibling task instead |
| `folder_required` | A task must always belong to a folder — if `update_task` tries to clear the folder, move to "Geral" instead |
| `cannot_delete_default_folder` | The "Geral" folder cannot be deleted; tell the user this is a system restriction |
| `task_not_found` / `folder_not_found` | Item may have been deleted; suggest calling `list_tasks` or `list_folders` to refresh |

---

## Workflow Patterns

**List pending tasks:**
1. `list_tasks` (filter `status: "pending"` or by folder)
2. Present in a clean list with due dates converted to BRT

**Create a task:**
1. `create_task` with title and optional `due_date`, `due_time`, `recurrence_type`, `recurrence_days` — omit `folder_id` unless the user explicitly named a folder
2. Confirm creation back to the user with the task title, recurrence, and folder

**Complete a task:**
1. `complete_task` with the task ID
2. If `parent_blocked_by_pending_subtasks` error: `get_task` → complete each subtask → retry parent

**Delete a task:**
1. Confirm with the user (show title + subtask count)
2. On confirmation: `delete_task`

---

## CRITICAL RULE — Destructive actions

Before calling `delete_task`, you MUST:
1. Send a confirmation message to the user with the task title and (if applicable) the number of subtasks that will be cascaded
2. Wait for explicit user confirmation in the next turn
3. Only then call `delete_task`

If the user's message is ambiguous ("apaga isso", "remove essa missão"), confirm which task. Do not assume.

NEVER call `delete_task` in the same turn the user requested deletion.
