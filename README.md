# Obsidian Notes Vault

Personal knowledge vault stored as plain-text files and synced with GitHub.

This repository is set up to share the actual vault content across machines while keeping machine-local Obsidian state out of Git. The goal is simple: when you pull on macOS or Windows, Git should mostly show real note changes, not UI noise.

## Vault Layout

- [CLI](./CLI)
- [Course](./Course)
- [Database](./Database)
- [LLM](./LLM)
- [Programming](./Programming)
- [Templates](./Templates)
- [assets](./assets)
- [projects](./projects)

Other folders such as `DL & ML` and `LLM & NLP` are part of the vault as well.

## What Is Shared In Git

These files are intentionally versioned because they represent real vault content or shared behavior:

- Notes and attachments: `*.md`, `*.base`, images, and other content files
- Shared vault settings in `.obsidian/`
- Community plugin code in `.obsidian/plugins/`
- Stable shared preferences such as `.obsidian/hotkeys.json`
- Useful shared plugin settings such as `.obsidian/plugins/colored-text/data.json`

## What Is Kept Local On Each Machine

These files are intentionally ignored because they change often and do not represent note content:

- `.obsidian/workspace.json`
- `.obsidian/workspaces.json`
- `.obsidian/workspace-mobile.json`
- `.obsidian/graph.json`
- `.obsidian/plugins/update-time-on-edit/data.json`
- `.trash/`
- `.DS_Store`, `Thumbs.db`, `desktop.ini`

## Why These Files Are Local

`workspace*.json` stores open tabs, pane layout, and similar session state. Obsidian's own docs call out `workspace.json` and `workspaces.json` as good Git ignore candidates because they update whenever you open files.

`graph.json` contains graph-view display state, including values such as zoom level. That is useful locally, but it creates unnecessary diffs between machines.

`update-time-on-edit/data.json` is local on purpose because the plugin stores a large `fileHashMap` cache in the same file as its settings. That cache changes as notes are edited, so sharing it through Git causes noisy diffs and merge conflicts.

## Community Plugin Policy

Current plugin handling in this vault:

- `dataview`: plugin files are tracked; no local data file is currently needed here
- `colored-text`: plugin files and `data.json` are tracked because the saved colors are stable shared preferences
- `update-time-on-edit`: plugin files are tracked, but `data.json` is local-only
- `obsidian-importer`: plugin files are tracked

This keeps plugin installation/version state shared through the repo, while excluding the one plugin settings file that behaves like a local cache.

## One-Time Setup On A New Machine

After pulling this repo on another machine:

1. Open the folder as a vault in Obsidian.
2. Let Obsidian load the shared `.obsidian` settings and community plugins from the repo.
3. Open `Update time on edit` settings and apply these local-only values:
   - `Date format`: `yyyy-MM-dd'T'HH:mm`
   - `Created property`: `created`
   - `Updated property`: `updated`
   - `Ignore for all updates`: `Templates/`, `README.md`
   - `Ignore for created property`: `Templates/`, `README.md`
4. Restart Obsidian once if a plugin asks for it.

The `Update time on edit` settings are the only part that must be set per machine. Everything else should come from Git.

## Daily Git Workflow

The intended workflow is:

1. Edit notes normally in Obsidian.
2. Pull or stash-and-pull as usual.
3. Commit real note changes and deliberate shared-setting changes.

After this setup is committed, the files that should stop bothering Git during normal use are:

- workspace layout files
- graph view state
- local cache from `update-time-on-edit`
- OS-specific junk files

If Git still shows changes in `.obsidian/`, they are much more likely to be real shared config changes that are worth reviewing before commit.

## Line Ending Policy

This repo includes [`.gitattributes`](./.gitattributes) to normalize line endings for Obsidian text files across macOS and Windows:

- `*.md`
- `*.json`
- `*.base`
- `*.canvas`

That reduces fake diffs caused only by CRLF/LF differences.

## Optional Advanced Setup

Obsidian also supports `Override config folder`, which lets each machine use a fully separate config directory. That is stronger isolation, but it also means plugin settings, hotkeys, and appearance no longer sync automatically through this repo.

This vault is intentionally set up for the middle ground:

- share content and shared vault behavior
- keep only the noisy local state out of Git

That gives a much smoother multi-machine Git workflow without giving up shared settings.
