# Agent Skills For Real Engineers

My agent skills that I use every day to do real engineering - not vibe coding.

If you want to keep up with changes to these skills, and any new ones I create, you can join ~60,000 other devs on my newsletter:

[Sign Up To The Newsletter](https://www.aihero.dev/s/skills-newsletter)

## Installation

All skills use the [Agent Skills](https://agentskills.io) open standard — `SKILL.md` files with YAML frontmatter. The file format is identical across agents; only the directory differs.

> **See [WORKFLOW.example](WORKFLOW.example)** for end-to-end examples showing how skills chain together for real projects (C multithreaded optimization, Forth compiler in Rust).

### Clone the repo

```bash
git clone https://github.com/abhinav20sep/ai-skills.git
```

### Install per agent

**Kilo Code CLI** — copies into `.kilo/skills/`:

```bash
mkdir -p .kilo/skills
cp -r ai-skills/<skill-name> .kilo/skills/
```

Also reads from `.opencode/skills/` and `~/.config/kilo/skills/` (global).

**OpenCode** — copies into `.opencode/skills/`:

```bash
mkdir -p .opencode/skills
cp -r ai-skills/<skill-name> .opencode/skills/
```

Also reads from `.claude/skills/` and `~/.config/opencode/skills/` (global).

**Claude Code** — copies into `.claude/skills/`:

```bash
mkdir -p .claude/skills
cp -r ai-skills/<skill-name> .claude/skills/
```

Also reads from `~/.claude/skills/` (global).

**Gemini CLI** — copies into `.gemini/skills/`, or use the install command:

```bash
# Option A: copy manually
mkdir -p .gemini/skills
cp -r ai-skills/<skill-name> .gemini/skills/

# Option B: install from repo
gemini skills install https://github.com/abhinav20sep/ai-skills.git

# Option C: link the whole repo
gemini skills link /path/to/ai-skills
```

Manage with `/skills list`, `/skills enable <name>`, `/skills disable <name>`.

**Kiro CLI** — copies into `.kiro/skills/`:

```bash
mkdir -p .kiro/skills
cp -r ai-skills/<skill-name> .kiro/skills/
```

Also reads from `~/.kiro/skills/` (global).

**Codex CLI** — copies into `.agents/skills/`:

```bash
mkdir -p .agents/skills
cp -r ai-skills/<skill-name> .agents/skills/
```

Also reads from `~/.agents/skills/` (global) and `/etc/codex/skills/` (system).

**Cursor** — copies into `.cursor/rules/` as `.mdc` files:

```bash
mkdir -p .cursor/rules
# Copy and rename SKILL.md to .mdc for each skill
for skill in scaffold-kernel-module setup-kernel-hooks migrate-unsafe-casts kernel-qa linux-threading linux-process linux-event-loop kernel-debugging linux-memory linux-networking linux-fileio linux-hardware; do
  cp "ai-skills/$skill/SKILL.md" ".cursor/rules/$skill.mdc"
done

Cursor uses `.mdc` files in `.cursor/rules/` instead of a `skills/` directory. The SKILL.md frontmatter (`name`, `description`) maps directly to Cursor's rule format. Also supports `.cursorrules` at project root (legacy).

### Install all skills at once

```bash
# Pick your agent's directory:
SKILL_DIR=.claude/skills    # or .kilo/skills, .opencode/skills, .gemini/skills, etc.

mkdir -p "$SKILL_DIR"
for skill in scaffold-kernel-module setup-kernel-hooks migrate-unsafe-casts kernel-qa linux-threading linux-process linux-event-loop kernel-debugging linux-memory linux-networking linux-fileio linux-hardware; do
  cp -r "ai-skills/$skill" "$SKILL_DIR/"
done

# For Cursor (uses .mdc files in .cursor/rules/):
mkdir -p .cursor/rules
for skill in scaffold-kernel-module setup-kernel-hooks migrate-unsafe-casts kernel-qa linux-threading linux-process linux-event-loop kernel-debugging linux-memory linux-networking linux-fileio linux-hardware; do
  cp "ai-skills/$skill/SKILL.md" ".cursor/rules/$skill.mdc"
done
```

Cursor uses `.mdc` files in `.cursor/rules/` instead of a `skills/` directory. The SKILL.md frontmatter (`name`, `description`) maps directly to Cursor's rule format. Also supports `.cursorrules` at project root (legacy).

### Install all skills at once

```bash
# Pick your agent's directory:
SKILL_DIR=.claude/skills    # or .kilo/skills, .opencode/skills, .gemini/skills, etc.

mkdir -p "$SKILL_DIR"
for skill in scaffold-kernel-module setup-kernel-hooks migrate-unsafe-casts kernel-qa linux-threading linux-process linux-event-loop kernel-debugging linux-memory linux-networking linux-fileio linux-hardware; do
  cp -r "ai-skills/$skill" "$SKILL_DIR/"
done

# For Cursor (uses .mdc files in .cursor/rules/):
mkdir -p .cursor/rules
for skill in scaffold-kernel-module setup-kernel-hooks migrate-unsafe-casts kernel-qa linux-threading linux-process linux-event-loop kernel-debugging linux-memory linux-networking linux-fileio linux-hardware; do
  cp "ai-skills/$skill/SKILL.md" ".cursor/rules/$skill.mdc"
done
```

### Install a single skill

```bash
# Example: install linux-threading for Claude Code
cp -r ai-skills/linux-threading .claude/skills/
```

### Global install (all projects)

```bash
# Kilo:    ~/.config/kilo/skills/
# OpenCode: ~/.config/opencode/skills/
# Claude:  ~/.claude/skills/
# Cursor:  ~/.cursor/rules/  (use .mdc extension)
# Gemini:  ~/.gemini/skills/
# Kiro:    ~/.kiro/skills/
# Codex:   ~/.agents/skills/

SKILL_DIR=~/.claude/skills   # adjust for your agent
mkdir -p "$SKILL_DIR"
cp -r ai-skills/<skill-name> "$SKILL_DIR/"
```

### Quick reference

| Agent | Skills directory | Global directory | Instructions file |
|---|---|---|---|
| **Kilo Code** | `.kilo/skills/` | `~/.config/kilo/skills/` | `AGENTS.md`, `CLAUDE.md` |
| **OpenCode** | `.opencode/skills/` | `~/.config/opencode/skills/` | `AGENTS.md` |
| **Claude Code** | `.claude/skills/` | `~/.claude/skills/` | `CLAUDE.md` |
| **Cursor** | `.cursor/rules/` (`.mdc` files) | `~/.cursor/rules/` | `.cursorrules` |
| **Gemini CLI** | `.gemini/skills/` | `~/.gemini/skills/` | `GEMINI.md` |
| **Kiro CLI** | `.kiro/skills/` | `~/.kiro/skills/` | `.kiro/steering/*.md` |
| **Codex CLI** | `.agents/skills/` | `~/.agents/skills/` | `AGENTS.md` |
| **Antigravity** | `.antigravity/skills/` * | `~/.antigravity/skills/` * | TBD |

\* Antigravity paths inferred from the Agent Skills standard — verify against your Antigravity config.

> **Note:** Pi could not be verified — no public documentation found for skill installation.

### Remote Linux machine (Cursor & Antigravity)

When working on a remote Linux machine via SSH (Cursor SSH Remote, Antigravity remote session, etc.), the skills must be installed **on the remote machine**, not locally.

**Option A: Clone the repo on the remote machine**

```bash
# SSH into your remote machine, then:
git clone https://github.com/abhinav20sep/ai-skills.git
cd <your-project>

# Cursor: copy rules into the project
mkdir -p .cursor/rules
for skill in ../ai-skills/*/; do
  cp "$skill/SKILL.md" ".cursor/rules/$(basename $skill).mdc"
done

# Antigravity: copy into the agent's instruction directory
# (check your Antigravity config for the correct path)
mkdir -p .antigravity/skills
cp -r ../ai-skills/* .antigravity/skills/
```

**Option B: Copy from local to remote via scp/rsync**

```bash
# From your local machine:
# Cursor
scp -r ai-skills/* user@remote:~/<project>/.cursor/rules/

# Antigravity
scp -r ai-skills/* user@remote:~/<project>/.antigravity/skills/
```

**Option C: Global install on the remote machine**

```bash
# On the remote machine, install globally so all projects pick them up:
# Cursor
mkdir -p ~/.cursor/rules
cp -r ai-skills/* ~/.cursor/rules/

# Antigravity
mkdir -p ~/.antigravity/skills
cp -r ai-skills/* ~/.antigravity/skills/
```

> **Antigravity note:** Antigravity's public documentation is not yet available. The paths above (`.antigravity/skills/`) are inferred from the agent skills standard. Check your Antigravity config for the correct directory. If it follows the Agent Skills open standard (like most agents), it will read from a `skills/` subdirectory within its config folder.

## Planning & Design

These skills help you think through problems before writing code.

- **to-prd** — Turn the current conversation context into a PRD and submit it as a GitHub issue. No interview — just synthesizes what you've already discussed.

  ```
  npx skills@latest add mattpocock/skills/to-prd
  ```

- **to-issues** — Break any plan, spec, or PRD into independently-grabbable GitHub issues using vertical slices.

  ```
  npx skills@latest add mattpocock/skills/to-issues
  ```

- **grill-me** — Get relentlessly interviewed about a plan or design until every branch of the decision tree is resolved.

  ```
  npx skills@latest add mattpocock/skills/grill-me
  ```

- **design-an-interface** — Generate multiple radically different interface designs for a module using parallel sub-agents.

  ```
  npx skills@latest add mattpocock/skills/design-an-interface
  ```

- **request-refactor-plan** — Create a detailed refactor plan with tiny commits via user interview, then file it as a GitHub issue.

  ```
  npx skills@latest add mattpocock/skills/request-refactor-plan
  ```

## Development

These skills help you write, refactor, and fix code.

- **tdd** — Test-driven development with a red-green-refactor loop. Builds features or fixes bugs one vertical slice at a time.

  ```
  npx skills@latest add mattpocock/skills/tdd
  ```

- **triage-issue** — Investigate a bug by exploring the codebase, identify the root cause, and file a GitHub issue with a TDD-based fix plan.

  ```
  npx skills@latest add mattpocock/skills/triage-issue
  ```

- **improve-codebase-architecture** — Find deepening opportunities in a codebase, informed by the domain language in `CONTEXT.md` and the decisions in `docs/adr/`.

  ```
  npx skills@latest add mattpocock/skills/improve-codebase-architecture
  ```

- **migrate-to-shoehorn** — Migrate test files from `as` type assertions to @total-typescript/shoehorn.

  ```
  npx skills@latest add mattpocock/skills/migrate-to-shoehorn
  ```

- **scaffold-exercises** — Create exercise directory structures with sections, problems, solutions, and explainers.

  ```
  npx skills@latest add mattpocock/skills/scaffold-exercises
  ```

## Linux & Systems Development

Skills for Linux kernel, driver, and userspace systems programming in C, Rust, and Zig.

- **scaffold-kernel-module** — Scaffold Linux kernel modules and drivers with correct boilerplate. Supports PCIe drivers, character devices, misc devices, platform drivers, Rust kernel modules, and Zig projects.

  ```
  npx skills@latest add abhinav20sep/ai-skills/scaffold-kernel-module
  ```

- **setup-kernel-hooks** — Set up native git hooks for C/Rust/Zig kernel projects with clang-format, checkpatch.pl, sparse, rustfmt, clippy, and zig fmt.

  ```
  npx skills@latest add abhinav20sep/ai-skills/setup-kernel-hooks
  ```

- **migrate-unsafe-casts** — Find and replace unsafe type casts with safer alternatives. Generates Coccinelle semantic patches for C, applies clippy fixes for Rust, and suggests Zig-safe patterns.

  ```
  npx skills@latest add abhinav20sep/ai-skills/migrate-unsafe-casts
  ```

- **kernel-qa** — Interactive QA session for kernel and driver bugs with kernel-specific awareness, reproduction strategies (QEMU, KUnit, fault injection), and issue filing.

  ```
  npx skills@latest add abhinav20sep/ai-skills/kernel-qa
  ```

- **linux-threading** — Multithreading patterns for Linux in C (pthreads), Rust (std::thread, Arc, Mutex, channels, Rayon), and Zig (std.Thread). Covers thread creation, synchronization, thread pools, lock-free patterns, and concurrency bug detection.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-threading
  ```

- **linux-process** — Process management and IPC patterns for Linux in C (fork/exec, pipes, shared memory, semaphores, message queues, Unix sockets, signals), Rust (std::process, nix crate), and Zig. Covers daemon processes, signal handling, and all POSIX IPC mechanisms.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-process
  ```

- **linux-event-loop** — Event-driven I/O patterns for Linux in C (epoll, select, poll, io_uring), Rust (mio, tokio), and Zig. Covers non-blocking I/O, event loops, io_uring, and high-performance network server design.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-event-loop
  ```

- **kernel-debugging** — Systematic Linux kernel debugging workflows using ftrace, perf, crash/kdump, KGDB, dynamic debug, and KASAN/KCSAN/KMSAN. Covers symptom-to-tool-to-interpretation patterns for kernel panics, hangs, performance issues, and memory corruption.

  ```
  npx skills@latest add abhinav20sep/ai-skills/kernel-debugging
  ```

- **linux-memory** — Linux memory management for kernel and userspace in C. Covers kernel allocators (kmalloc, vmalloc, slab caches, mempools, GFP flags), managed allocations (devm_*), DMA mapping API, userspace mmap, memory barriers, and OOM handling.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-memory
  ```

- **linux-networking** — Linux network programming in C (sockets, raw sockets, netlink), Rust (std::net, tokio), and Zig. Covers TCP/UDP client/server, raw sockets, netlink sockets, TCP tuning, and TLS integration.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-networking
  ```

- **linux-fileio** — Linux file I/O patterns in C (direct I/O, async I/O, file locking, sendfile, mmap file I/O), Rust, and Zig. Covers O_DIRECT, io_uring for files, flock, fcntl locks, zero-copy transfer, and memory-mapped file access.

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-fileio
  ```

- **linux-hardware** — Hardware access patterns for Linux device drivers in C. Covers MMIO register access, port I/O, device tree parsing, ACPI, regmap API, and interrupt handling (threaded IRQs, IRQ domains).

  ```
  npx skills@latest add abhinav20sep/ai-skills/linux-hardware
  ```

## Tooling & Setup

- **setup-pre-commit** — Set up Husky pre-commit hooks with lint-staged, Prettier, type checking, and tests.

  ```
  npx skills@latest add mattpocock/skills/setup-pre-commit
  ```

- **git-guardrails-claude-code** — Set up Claude Code hooks to block dangerous git commands (push, reset --hard, clean, etc.) before they execute.

  ```
  npx skills@latest add mattpocock/skills/git-guardrails-claude-code
  ```

## Writing & Knowledge

- **write-a-skill** — Create new skills with proper structure, progressive disclosure, and bundled resources.

  ```
  npx skills@latest add mattpocock/skills/write-a-skill
  ```

- **edit-article** — Edit and improve articles by restructuring sections, improving clarity, and tightening prose.

  ```
  npx skills@latest add mattpocock/skills/edit-article
  ```

- **ubiquitous-language** — Extract a DDD-style ubiquitous language glossary from the current conversation.

  ```
  npx skills@latest add mattpocock/skills/ubiquitous-language
  ```

- **obsidian-vault** — Search, create, and manage notes in an Obsidian vault with wikilinks and index notes.

  ```
  npx skills@latest add mattpocock/skills/obsidian-vault
  ```
