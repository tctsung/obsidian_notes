---
created: 2026-10-03T19:19
updated: 2026-10-03T19:42
---
# Obsidian Notes

My personal knowledge vault. Plain Markdown, synced via GitHub. 
Some notes are published in personal website: https://tctsung.github.io/#/blog

## Layout

| Folder | Contents |
|---|---|
| `CLI/` | Shell, Git, SSH, terminal tools |
| `Course/` | Course notes (e.g. CS336) |
| `Database/` | Databases and query languages |
| `DL & ML/` | Deep learning / ML concepts and frameworks |
| `LLM & NLP/` | LLM concepts (RAG, prompting, tokenization) and frameworks |
| `Programming/` | Language notes (Python, ...) |
| `projects/` | Project notes |
| `Templates/` | Obsidian templates |
| `_private/` | **Gitignored.** Machine-local sensitive notes |

Images live in an `assets/` folder next to the notes that use them.

## Ignored content

| Ignored | Why |
|---|---|
| `_private/` | Sensitive notes that must never reach the public repo. Local to each machine: work credentials, account IDs, and hostnames on the work laptop; personal secrets on the personal one |
| `.obsidian/workspace*.json`, `.obsidian/graph.json` | Open tabs, pane layout, and graph view state. They change on every click |
| `.obsidian/plugins/update-time-on-edit/data.json` | The plugin keeps a file-hash cache here that changes on every edit |
| `.trash/`, `.DS_Store`, `Thumbs.db`, `desktop.ini` | Deleted files and OS junk |

Everything else is tracked: notes, attachments, and shared `.obsidian/` settings (plugins, hotkeys, `colored-text` colors). `.gitattributes` forces LF line endings so macOS and Windows don't create fake diffs.

If a public note needs a secret (e.g. an SSH config), use a placeholder like `<account-id>` and keep the real value in `_private/`. Renaming a file doesn't hide it.

## New machine setup

1. Clone, then open the folder as a vault in Obsidian. Shared settings and plugins load from the repo.
2. Set up **Update time on edit**. Its settings are local-only, so enter them by hand:
   - Date format: `yyyy-MM-dd'T'HH:mm`
   - Created / Updated property: `created` / `updated`
   - Ignore for all updates and for created: `Templates/`, `README.md`, `_private/`
3. Create `_private/` if you need it on this machine.
