# realm-markdown-memory

Markdown memory under the shared `Memory` label. Agents keep their memory as markdown files —
Claude Code's memory directory, OpenClaw's `MEMORY.md` and daily logs, a Cline/Roo memory bank,
plain notes — and this realm reads those files live, so one Virtual Cypher query answers from every
file-based memory alongside the assistant's own semantic memory and any memory service.

```cypher
MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:Memory)          -- every memory system
RETURN m.source, m.file, m.text, r.score, r.mode ORDER BY m.source

MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:MarkdownMemory)  -- every file-based one
MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:OpenClawMemory)  -- one system alone
```

Every node carries `:Memory`, `:MarkdownMemory`, its system's label and the realm label `:Markdown`.

## The systems

| Label | What it reads | Folder under the world's `data/memory/` |
|---|---|---|
| `ClaudeMemory` | Claude Code's auto-memory: one frontmatter file per fact, plus `MEMORY.md` | `claude/` |
| `OpenClawMemory` | OpenClaw's `MEMORY.md` (durable facts) and `memory/YYYY-MM-DD.md` (daily logs) | `openclaw/` |
| `MemoryBankMemory` | a Cline / Roo / Kilo memory bank: `projectbrief`, `activeContext`, `progress`, … | `memory-bank/` |
| `NotesMemory` | any markdown notes: an Obsidian vault, a notes folder, daily notes | `notes/` |

A folder is either the memory itself or a symlink to where the agent keeps it. The files walk is
confined to the world's `data/` sandbox, so a symlink is followed only when its target is declared
in the world's `world.yml`:

```yaml
filesTrustedRoots:
  - /Users/you/.claude/projects/-Users-you-dev-myproject/memory
  - /Users/you/.openclaw/workspace
```

```bash
cd <world>/data/memory
ln -s ~/.claude/projects/-Users-you-dev-myproject/memory claude
ln -s ~/.openclaw/workspace openclaw
```

Nothing is ingested or indexed: a file edited on disk is what the next query reads.

## How it answers

Retrieval is lexical. The Concept's text is searched in the files; a file whose content contains it
is one memory, `m.text` is the matching passage with a line of context, `m.file` and `m.url` say
where it came from, and `r.score` is a neutral 1.0 meaning "matched". `via:'agentic-rag'` expands
the Concept into probe terms first, so "equity compensation" can find the note that says "option
grant".

The data contract is `Memory`'s (realm-spec `VIRTUAL_CYPHER.md` §6.8): `id` is the file path,
`source` names the system, `created` / `revised` are the file's own times, `confidence` is 1.0
and `status` is ACTIVE — a markdown memory is current by being on disk. Instruction files such as
`CLAUDE.md`, `AGENTS.md` and `SOUL.md` are deliberately not memories; put them nowhere under
`data/memory`.

## The ladder

The shared label is a migration path, and this realm is one rung of it.

- **Mirror** — install the realm and link the agent's memory folder under `data/memory/`; its files
  answer under `:Memory` beside the assistant's own from the next query.
- **Compare** — `memory_bakeoff {topic}` lists what each system recalls, one row per memory with
  `system` naming it (`claude`, `openclaw`, `memory-bank`, `notes`); `memory_systems` counts per system.
- **Promote** — `gateway.memory.promote({text, source, id})` copies a fact into the assistant's own
  memory; the row then reads `status: PROMOTED` and leaves plain `:Memory` reads.
- **Retire** — uninstall the realm, or unlink one folder; every `:Memory` query keeps answering from
  the systems that remain.

`source` and `status` are stamped by each producer's `compute:` block, one per system.
