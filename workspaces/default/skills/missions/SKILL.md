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

## Available Tools

| Tool | When to use |
|------|-------------|
| `list_folders` | List all folders; use to discover folder IDs before creating tasks |
| `create_folder` | Create a new folder/category for grouping tasks |
| `get_folder` | Get details of a specific folder including its tasks |
| `update_folder` | Rename or modify a folder |
| `list_tasks` | List tasks; supports filtering by folder, status, due date |
| `create_task` | Create a new task (with optional folder, due date, recurrence, subtasks) |
| `get_task` | Get full details of a single task including subtasks |
| `update_task` | Edit title, description, folder, due date, recurrence — NOT for marking done |
| `complete_task` | Mark a task as done — the ONLY way to change status to "done" |
| `delete_task` | Permanently delete a task and all its subtasks (irreversible — confirm first) |

> `delete_folder` is **not available** via BMO. Folders must be deleted through the frontend app.

---

## Business Rules

### Subtasks
- Subtasks are supported only **1 level deep** — you cannot create a subtask of a subtask.
- To add a subtask, use `create_task` with a `parent_id` pointing to the parent task.

### Recurring Tasks
- Recurring tasks **require** `due_at` — always ask the user for a start date/time if missing.
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
  [0–6]         → if weekly (0 = Monday, 6 = Sunday)
  [1–31]        → if monthly (day of month)
```

**Examples:**
- "Todo dia" → `recurrence_type: "daily"`, `recurrence_days: null`
- "Toda segunda e quarta" → `recurrence_type: "weekly"`, `recurrence_days: [0, 2]`
- "Todo dia 15" → `recurrence_type: "monthly"`, `recurrence_days: [15]`
- "Toda sexta" → `recurrence_type: "weekly"`, `recurrence_days: [4]`

---

## Dates & Timezones

- All `due_at` values must be **ISO 8601 UTC** (e.g., `2026-05-13T21:00:00Z`).
- The user is in **America/Sao_Paulo (UTC-3)**. Convert accordingly:
  - "amanhã às 18h" = tomorrow 18:00 BRT = tomorrow 21:00 UTC
- When displaying dates back to the user, convert to local time (UTC-3).

---

## Error Handling

| Error code | How to handle |
|------------|---------------|
| `parent_blocked_by_pending_subtasks` | Explain to user; call `get_task` to list which subtasks are pending, then offer to complete them one by one |
| `recurrence_requires_due_at` | Ask the user for the start date/time before retrying |
| `subtask_depth_exceeded` | Explain the 1-level limit; offer to create as a sibling task instead |
| `task_not_found` / `folder_not_found` | Item may have been deleted; suggest calling `list_tasks` or `list_folders` to refresh |

---

## Workflow Patterns

**List pending tasks:**
1. `list_tasks` (filter `status: "pending"` or by folder)
2. Present in a clean list with due dates converted to BRT

**Create a task:**
1. If a folder is needed, call `list_folders` first to get the `folder_id`
2. `create_task` with title, optional `folder_id`, `due_at`, recurrence fields
3. Confirm creation back to the user with the task title

**Complete a task:**
1. `complete_task` with the task ID
2. If `parent_blocked_by_pending_subtasks` error: `get_task` → complete each subtask → retry parent

**Delete a task:**
1. Confirm with the user (show title + subtask count)
2. On confirmation: `delete_task`
