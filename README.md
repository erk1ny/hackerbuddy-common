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

Agents interact with tools via a start/read/stop lifecycle. The `knowledge.md` content is injected into skill instructions wherever `@tool:name` appears, giving agents calibration patterns and decision logic inline.

**tool.yaml schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name (lowercase-kebab-case) |
| `description` | string | Yes | What the tool does |
| `binary` | string | Yes | Command name (must be in PATH) |
| `categories` | list[string] | Yes | Tool categories |
| `intensity` | int (1-5) | Yes | Work queue lane weight |
| `install` | string | Yes | Install instructions (shown when binary missing) |
| `scope_extraction` | object | No | `{flag, pattern}` — extracts target hostname from CLI args |
| `output_flags` | string | No | Flags for structured output format |
| `default_timeout` | int | No | Timeout in seconds (default 300) |

### Skills

Each skill is a directory under `skills/` containing:

- `skill.yaml` — skill metadata and configuration
- `skill.md` — markdown instructions that agents follow step by step

**Two types:**
- **Atomic** — single focused checks. Cannot reference other skills.
- **Full** — multi-phase procedures that compose atomic skills via `@skill-name` references.

**skill.yaml schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name (lowercase-kebab-case) |
| `description` | string | Yes | What the skill checks |
| `type` | string | Yes | `"atomic"` or `"full"` |
| `tags` | list[string] | Yes | For filtering/discovery |
| `mode` | string | Yes | Primary agent mode (discover, research, exploit, verify, report) |
| `version` | string | No | Per-skill version tracking (default "0.0.0") |
| `tools` | list[string] | No | Required tool names (cross-validated against tools/) |
| `depends_on` | list[string] | No | Atomic skill names (must be empty for atomic type) |
| `intensity` | int (1-5) | No | Work queue lane weight (default 3) |
| `inputs` | list[object] | No | Input parameters (each: name, type, required, default, description) |

**Reference syntax in skill.md:**
- `@tool:name` — replaced with the tool's `knowledge.md` content at resolution time
- `@skill-name` — replaced with the atomic skill's `skill.md` instructions (full skills only, must match `depends_on`)

**Mandatory reporting:** agents must call `report_skill_result` after every skill with one of: `found`, `not_found`, `partial`, `variant`, `error`.

## How to add a new tool

1. Create `tools/<name>/` directory (lowercase-kebab-case)
2. Add `tool.yaml` with all required fields
3. Add `knowledge.md` with: what it does, core flags, calibration patterns, output format
4. Bump `manifest.json` version
5. Open a PR

## How to add a new skill

1. Create `skills/<name>/` directory (lowercase-kebab-case)
2. Add `skill.yaml` with all required fields
3. Add `skill.md` with: objective, step-by-step procedure, expected outcomes, output format
4. Use `@tool:name` where the agent needs tool knowledge injected
5. If full type: add `depends_on` entries and use `@skill-name` references in skill.md
6. Bump `manifest.json` version
7. Open a PR

## Testing locally

1. Copy your tool/skill directory to `~/.config/hackerbuddy/{tools,skills}/user/`
2. Run `hackerbuddy chat`
3. Startup validation catches structural errors (bad yaml, missing deps, type mismatches)
4. Use `list_skills` / `list_tools` MCP tools to confirm visibility
5. Run a test workflow to verify instructions make sense to agents
6. Move from `user/` to the community repo once validated
