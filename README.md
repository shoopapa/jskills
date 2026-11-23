# jskills

Global agent skills with `~/.agents/skills` as the single source of truth across **Claude Code**, **Codex**, **Antigravity**, and **Gemini CLI**.

## Included Skills

| Skill | Purpose |
| :--- | :--- |
| `autopilot` | Keep a PR merge-ready by triaging comments, resolving conflicts, and fixing CI in a loop |
| `babysit-pr` | Monitor a pull request through review and CI |
| `council` | Multi-model review/consensus workflow |
| `design-director` | Art direction and rendered visual review for UI, slides, reports, and visual artifacts |
| `epic` | End-to-end epic planning and execution workflow |
| `file-pr` | File a concise pull request |
| `full-send` | Autonomous end-to-end execution to a merged PR |
| `full-stop` | Immediate stop protocol |
| `leaving-pr-comments` | Guidelines for leaving concise, constructive GitHub PR comments |
| `modernize` | Rebase and update older PRs |
| `sol-loop` | Iterative code review loop |
| `split-to-prs` | Safely split a large branch or diff into smaller reviewable PRs |
| `takeover` | Take over an existing PR, branch, or agent thread |

---

## Linking Strategy

Rather than symlinking entire `skills/` directories, this setup uses **per-skill symlinks** from `~/.agents/skills/<skill>` into each tool's global skills directory:

| Tool | Target Directory |
| :--- | :--- |
| Claude Code | `~/.claude/skills` |
| Codex | `~/.codex/skills` |
| Antigravity | `~/.gemini/config/skills` |
| Gemini CLI | `~/.gemini/skills` |

### Why per-skill symlinks?
- **Preserves tool-managed folders:** Codex stores built-in skills in `~/.codex/skills/.system/`. Per-skill symlinks keep `.system/` isolated instead of polluting `~/.agents/skills/`.
- **Non-destructive:** Existing non-symlink directories in target folders are never overwritten.
- **Self-cleaning:** Removing a skill from `~/.agents/skills/` automatically prunes its broken symlinks across all targets on the next shell start (or when running `sync_agent_skills`).
- **Shell-portable:** Works in both `zsh` and `bash` (important if `~/.zshrc` is synced to `~/.bashrc` / `~/.profile`).

---

## Setup on a New Machine

### 1. Clone and populate `~/.agents/skills`

```bash
git clone https://github.com/shoopapa/jskills.git ~/code/jskills
mkdir -p ~/.agents
ln -sfn ~/code/jskills/skills ~/.agents/skills
# Or copy instead of symlinking if you prefer a decoupled local copy:
# cp -R ~/code/jskills/skills ~/.agents/skills
```

### 2. Add `sync_agent_skills` to `~/.zshrc`

```sh
# ── Sync global agent skills from ~/.agents/skills ────────────────────────────
# Keeps ~/.agents/skills as the single source of truth across Claude Code,
# Codex, Antigravity (~/.gemini/config/skills), and Gemini CLI (~/.gemini/skills).
# Uses per-skill symlinks so tool-managed folders like ~/.codex/skills/.system
# stay isolated.
sync_agent_skills() {
  local src="$HOME/.agents/skills"
  [ -d "$src" ] || return 0
  [ -n "${ZSH_VERSION:-}" ] && setopt local_options null_glob

  local target skill name dest link_target
  for target in \
    "$HOME/.claude/skills" \
    "$HOME/.codex/skills" \
    "$HOME/.gemini/config/skills" \
    "$HOME/.gemini/skills"
  do
    mkdir -p "$target" 2>/dev/null || continue

    # Symlink each skill from ~/.agents/skills
    for skill in "$src"/*; do
      [ -d "$skill" ] || continue
      name="${skill##*/}"
      dest="$target/$name"
      if [ -L "$dest" ]; then
        [ "$(readlink "$dest")" = "$skill" ] || ln -sfn "$skill" "$dest"
      elif [ ! -e "$dest" ]; then
        ln -s "$skill" "$dest"
      fi
    done

    # Prune broken symlinks that used to point into ~/.agents/skills
    for dest in "$target"/*; do
      [ -L "$dest" ] || continue
      link_target="$(readlink "$dest")"
      case "$link_target" in
        "$src"/*) [ -e "$dest" ] || rm -f "$dest" ;;
      esac
    done
  done
}
sync_agent_skills
```

### 3. Apply

```bash
source ~/.zshrc
```

Whenever you add, pull, or remove skills in `~/.agents/skills`, either open a new shell or run `sync_agent_skills`.
