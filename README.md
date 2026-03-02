# hackerbuddy-common

Community tools and skills for [Hackerbuddy](https://github.com/erk1ny/hackerbuddy) — an AI-native terminal environment for web vulnerability research.

## What's in this repo

- **Tools** (`tools/`) — registered external CLI binaries + knowledge documents that teach agents how to use them effectively
- **Skills** (`skills/`) — prompt-based markdown instructions that encode vulnerability patterns and research procedures

Hackerbuddy downloads this repo automatically on launch and installs tools and skills to `~/.config/hackerbuddy/{tools,skills}/community/`.

## File formats

### Tools

Each tool is a directory under `tools/` containing:

- `tool.yaml` — tool metadata and configuration
- `knowledge.md` — natural language document teaching the agent how to use the tool

**tool.yaml schema:**

```yaml
name: ffuf                          # Must match directory name (lowercase-kebab-case)
description: Fast web fuzzer        # What the tool does
binary: ffuf                        # Command name (must be in PATH)
categories: [fuzzing, discovery]    # Tool categories
intensity: 5                        # 1-5 for work queue lane assignment
scope_extraction:                   # How to extract target hostname from args
  flag: "-u"                        # CLI flag containing target URL
  pattern: "https?://([^/:]+)"      # Regex to extract hostname
output_flags: "-o {output_file} -of json"  # Flags for structured output
default_timeout: 300                # Timeout in seconds
install: |                          # Install instructions (shown when binary missing)
  brew install ffuf
```

Set `scope_extraction` to null/omit for tools that don't target URLs (local analysis only).

### Skills

Each skill is a directory under `skills/` containing:

- `skill.yaml` — skill metadata and configuration
- `skill.md` — markdown instructions that agents follow

**Two types:**
- **Atomic** — single focused checks. Cannot reference other skills.
- **Full** — multi-phase procedures that compose atomic skills via `@skill-name` references.

**skill.yaml schema:**

```yaml
name: check-framing                 # Must match directory name (lowercase-kebab-case)
version: "1.0.0"                    # Per-skill version tracking
description: Check for clickjacking # What the skill checks
type: atomic                        # "atomic" or "full"
tags: [clickjacking, headers]       # For filtering/discovery
tools: []                           # Required tool names (cross-validated)
mode: discover                      # Primary agent mode
depends_on: []                      # Atomic: must be empty. Full: list of atomic skill names
intensity: 3                        # 1-5 for work queue lane assignment
```

**Reference syntax in skill.md:**
- `@skill-name` — references another atomic skill (full skills only). Must match `depends_on`.
- `@tool:name` — references a tool's knowledge. Both types can use this.

## How to add a new tool

1. Create `tools/<name>/` directory (lowercase-kebab-case)
2. Add `tool.yaml` with all required fields
3. Add `knowledge.md` with usage patterns and decision logic
4. Bump `manifest.json` version
5. Open a PR

## How to add a new skill

1. Create `skills/<name>/` directory (lowercase-kebab-case)
2. Add `skill.yaml` with all required fields
3. Add `skill.md` with detection/research procedures
4. If full type: add `depends_on` entries and use `@skill-name` references in skill.md
5. Bump `manifest.json` version
6. Open a PR

## Testing locally

1. Copy your tool/skill directory to `~/.config/hackerbuddy/{tools,skills}/user/`
2. Run `hackerbuddy chat`
3. Startup validation catches structural errors (bad yaml, missing deps, type mismatches)
4. Use `list_skills` / `list_tools` MCP tools to confirm visibility
5. Run a test workflow to verify instructions make sense to agents
6. Move from `user/` to the community repo once validated
