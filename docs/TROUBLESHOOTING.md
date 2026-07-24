# Troubleshooting

## Quick reference for common install/setup problems

Four issues account for most support questions. Check these first:

### 1. `Hooks use a matcher + hooks array` error in `.claude/settings.json`

**You're on a pre-v2.2.3 release.** v2.2.1 and v2.2.2 shipped a broken hook schema — flat `{matcher, command, timeout}` entries without the required nested `hooks: []` array, timeouts in milliseconds instead of seconds, and a `PreCommit` event that isn't a real Claude Code event. PR #208 (shipped in v2.2.3) rewrote the generator to emit the correct v1.x+ schema.

**Fix:**

```bash
pip install --upgrade Bitroot Code Review   # → v2.2.4 or later
cd /path/to/your/project
bitroot-code-review-kit install                 # rewrites .claude/settings.json
```

The re-install merge-replaces the entire broken `hooks` block with the new nested format and drops a real git pre-commit hook into the hooks directory resolved via `git rev-parse --git-path hooks` — typically `.git/hooks/pre-commit`, but linked worktrees and `core.hooksPath` (husky) setups are handled too. That's where "check before commit" lives in v2.2.3+, not in Claude Code settings.

Valid Claude Code hook events are: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `SessionStart`, `SessionEnd`, `PreCompact`, `Notification`. There is no `PreCommit`.

### 2. `Bitroot Code Review: command not found` after `pip install`

`pip install` put the console script into a `bin/` directory that isn't on your `$PATH`. Four fixes, in order of recommendation:

**Option 1 — Use `pipx` (cleanest):**

```bash
pip uninstall Bitroot Code Review
pipx install bitroot-code-review-kit
```

`pipx` installs CLI tools in an isolated venv. If the command is not found afterwards, run `pipx ensurepath` or add `~/.local/bin` to your PATH.

**Option 2 — Use `uvx` (no install needed):**

```bash
uvx bitroot-code-review-kit install
uvx bitroot-code-review-kit build
```

**Option 3 — Run it as a Python module (always works):**

```bash
python -m code_review_graph install
python -m code_review_graph build
```

**Option 4 — Fix PATH manually:**

```bash
pip show Bitroot Code Review | grep Location
# Find the sibling `bin/` directory; on macOS user installs this is
# typically ~/Library/Python/3.X/bin. Add it to your shell rc:
echo 'export PATH="$HOME/Library/Python/3.12/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 3. Is Bitroot Code Review project-scoped or user-scoped?

**Both** — four different pieces, each scoped differently:

| Piece                         | Scope          | Where                                                            |
|-------------------------------|----------------|------------------------------------------------------------------|
| The Python package            | User-scoped    | Install once via `pip`/`pipx`/`uvx`                              |
| The graph database            | Project-scoped | `.code-review-graph/graph.db` inside each project                |
| MCP server config (`.mcp.json`) | Project-scoped | Claude Code launches one MCP server per project, with `cwd=<project>` |
| Multi-repo registry           | User-scoped    | `~/.code-review-graph/registry.json` (only for `cross_repo_search`) |

**TL;DR**: install the tool **once**, then run `bitroot-code-review-kit install && bitroot-code-review-kit build` inside **each** project you want graph-aware reviews in.

### 4. Using a venv? You must update `settings.json` manually

Claude Code hooks and MCP tool paths in `.claude/settings.json` are **hardcoded at install time**. If you switch to (or create) a virtual environment after running `bitroot-code-review-kit install`, the paths will still point to the old interpreter and the server will silently fail or use the wrong Python.

**Fix — update the `command`/`args` in `.mcp.json` and any hook commands in `.claude/settings.json` to match your venv:**

```json
// .mcp.json — point to your venv's Python or uvx inside the venv
{
  "mcpServers": {
    "Bitroot Code Review": {
      "command": "/path/to/your/venv/bin/uvx",
      "args": ["Bitroot Code Review", "serve"]
    }
  }
}
```

Or simply re-run `bitroot-code-review-kit install` **from within the activated venv** so the paths are regenerated correctly:

```bash
source .venv/bin/activate          # activate your venv first
bitroot-code-review-kit install          # rewrites .mcp.json and hook paths
```

Then fully quit and reopen Claude Code so it picks up the new config.

### 5. "I built the graph but Claude Code doesn't see it in a new session"

Most likely causes, ranked:

1. **You didn't restart Claude Code after `install`.** Claude Code reads `.mcp.json` at startup — if you ran `install` in one session, fully quit and reopen Claude Code for the MCP server to register.
2. **New session's `cwd` is a different directory.** The MCP server is launched with `cwd=<project>` and it reads `.code-review-graph/graph.db` from there. If your new session opened in a parent folder or a different project, it won't find the graph you built.
3. **You ran `build` but not `install`.** `build` creates `graph.db`; `install` is what registers the MCP server with Claude Code via `.mcp.json`. You need both.
4. **MCP server is crashing on startup.** Run `/mcp` inside Claude Code to see server status, or check `~/Library/Logs/Claude/mcp*.log` on macOS.

**Quick checklist:**

```bash
cd /path/to/your/project
bitroot-code-review-kit status    # should print Files/Nodes/Edges from the built graph
ls .mcp.json                # should exist
cat .mcp.json               # should reference `bitroot-code-review-kit serve`
# then: fully quit Claude Code and reopen it inside this project
```

If `status` shows the graph but `/mcp` in the new session doesn't list `Bitroot Code Review`, the `.mcp.json` isn't in the session's `cwd` — re-run `bitroot-code-review-kit install` from the correct project root.

---

## Database lock errors
The graph uses SQLite with WAL mode. If you see lock errors:
- Ensure only one build process runs at a time
- The database auto-recovers; just retry
- Delete `.code-review-graph/graph.db-wal` and `.code-review-graph/graph.db-shm` if corrupt

## Large repositories (>10k files)
- First build may take 30-60 seconds
- Subsequent incremental updates are fast (<2s)
- Add more ignore patterns to `.code-review-graphignore`:
  ```
  generated/**
  vendor/**
  *.min.js
  ```

## Missing nodes after build
- Check that the file's language is supported (see [FEATURES.md](FEATURES.md))
- Check that the file isn't matched by an ignore pattern
- Run with `full_rebuild=True` to force a complete re-parse

## Graph seems stale
- Hooks auto-update on edit/commit
- If stale, run `/bitroot-code-review-kit:build-graph` manually
- Check that hooks are configured in `.claude/settings.json` (re-run `bitroot-code-review-kit install` to regenerate)

## Embeddings not working
- Install with: `pip install "bitroot-code-review-kit[embeddings]"`
- Run `embed_graph_tool` to compute vectors
- First embedding run downloads the model (~90MB, one time)

## MCP server won't start
- Verify `uv` is installed (`uv --version`; install with `pip install uv` or `brew install uv`)
- Check that `uvx bitroot-code-review-kit serve` runs without errors
- If using a custom `.mcp.json`, ensure it uses `"command": "uvx"` with `"args": ["Bitroot Code Review", "serve"]`
- Re-run `bitroot-code-review-kit install` to regenerate the config

## Windows / WSL

- Upgrade to v2.3.6+ if `daemon status` crashes with WinError 87 (#511) or CLI `detect-changes` maps 0 functions on Windows (#528) — both are fixed there
- Use forward slashes in paths when passing `repo_root` to MCP tools
- In WSL, ensure `uv` is installed inside WSL (not the Windows version): `curl -LsSf https://astral.sh/uv/install.sh | sh`
- If `uv` is not found after install, add `~/.cargo/bin` to your PATH
- File watching (`bitroot-code-review-kit watch`) may have delays on WSL1 due to filesystem event limitations; WSL2 is recommended
- On Windows native (non-WSL), long path support may need to be enabled: `git config --system core.longpaths true`

## Community detection requires igraph

- Install with: `pip install "bitroot-code-review-kit[communities]"`
- Without igraph, community detection falls back to file-based grouping (less precise but functional)

## Wiki generation with LLM summaries

- Install with: `pip install "bitroot-code-review-kit[wiki]"`
- Requires a running Ollama instance for LLM-powered summaries
- Without Ollama, wiki pages are generated with structural information only (no prose summaries)

## Optional dependency groups

If a tool returns an ImportError, install the relevant optional group:
- `pip install "bitroot-code-review-kit[embeddings]"` for semantic search
- `pip install "bitroot-code-review-kit[google-embeddings]"` for Google Gemini embeddings
- OpenAI-compatible and MiniMax embeddings use stdlib HTTP clients and require only their environment variables
- `pip install "bitroot-code-review-kit[communities]"` for igraph-based community detection
- `pip install "bitroot-code-review-kit[enrichment]"` for Python call-resolution enrichment via Jedi
- `pip install "bitroot-code-review-kit[eval]"` for evaluation benchmarks (matplotlib)
- `pip install "bitroot-code-review-kit[wiki]"` for wiki LLM summaries (ollama)
- `pip install "bitroot-code-review-kit[all]"` for everything
