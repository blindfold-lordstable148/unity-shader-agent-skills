# Unity Shader Agent Skills

**7 curated agent skills** for Unity shader development targeting mobile GPUs and WebGL browsers. Built for [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code) using the `SKILL.md` format, but the knowledge is portable to any AI coding assistant.

Each skill distills 150+ verified resources (developer blogs, GitHub repositories, code snippets, and technical guides) into actionable instructions an AI agent can follow when writing, reviewing, or optimizing Unity shaders.

## Skills

| Skill | Description | Depth |
|-------|-------------|-------|
| [`mobile-shader-optimization`](skills/mobile-shader-optimization/SKILL.md) | GPU architecture, precision types, fillrate, overdraw, baked lighting, LOD | Comprehensive |
| [`water-fluid-shaders`](skills/water-fluid-shaders/SKILL.md) | Mobile-optimized water with depth, foam, waves, refraction, Gerstner | Comprehensive |
| [`mobile-post-processing`](skills/mobile-post-processing/SKILL.md) | Single-pass post-processing, URP Renderer Features, mobile-safe effects | Standard |
| [`urp-hlsl-templates`](skills/urp-hlsl-templates/SKILL.md) | Production-ready URP shader code templates with SRP Batcher compatibility | Comprehensive |
| [`webgl-shader-constraints`](skills/webgl-shader-constraints/SKILL.md) | WebGL 1.0/2.0 restrictions, variant stripping, loop/indexing rules | Standard |
| [`shader-graph-best-practices`](skills/shader-graph-best-practices/SKILL.md) | Node budgets, custom lighting, sub-graphs, precision, mobile workflows | Standard |
| [`texture-packing-variant-stripping`](skills/texture-packing-variant-stripping/SKILL.md) | Channel packing, variant reduction, shader_feature vs multi_compile | Standard |

## Install

### One command

```bash
npx skills add adevra/unity-shader-agent-skills
```

The CLI walks you through it interactively — pick your agent (Claude Code, Cursor, etc.), choose project or global scope, and select which skills you want.

### Install specific skills

```bash
npx skills add adevra/unity-shader-agent-skills --skill mobile-shader-optimization --skill water-fluid-shaders
```

### Target a specific agent

```bash
npx skills add adevra/unity-shader-agent-skills -a claude-code
```

### Non-interactive (install everything)

```bash
npx skills add adevra/unity-shader-agent-skills --all -y
```

### Project-scoped vs User-scoped

```bash
# Project-scoped (default) — installs to ./<agent>/skills/
npx skills add adevra/unity-shader-agent-skills

# User-scoped (global) — installs to ~/<agent>/skills/, applies to all projects
npx skills add adevra/unity-shader-agent-skills -g
```

| Scope | Path | Applies to |
|-------|------|------------|
| **Project** | `.claude/skills/`, `.cursor/skills/`, etc. | This project only |
| **User** (`-g`) | `~/.claude/skills/`, `~/.cursor/skills/`, etc. | All your projects |

### Manual install (alternative)

```bash
git clone https://github.com/adevra/unity-shader-agent-skills.git
cp -r unity-shader-agent-skills/skills/ /path/to/your/project/.claude/skills/
```

## Supported Agents

The `SKILL.md` files are plain markdown with structured sections. The `npx skills add` CLI supports:

- **Claude Code** — `.claude/skills/`
- **Cursor** — `.cursor/skills/`
- **Windsurf** — `.windsurf/skills/`
- **Custom agents** — Feed as system prompts or context documents

## What's Inside Each Skill

Every `SKILL.md` follows a consistent structure:

1. **Trigger description** — When the agent should activate this skill
2. **Core rules** — Hard constraints the agent must follow
3. **Code templates** — Copy-paste ready ShaderLab/HLSL or Shader Graph guidance
4. **Optimization checklist** — Concrete steps to validate shader quality
5. **Reference links** — Verified external resources for deeper reading

## Sources

All skills are distilled from verified, high-quality resources including:

- **Catlike Coding**, **Cyanilux**, **Daniel Ilett**, **NedMakesGames** — top shader educators
- **ColinLeung-NiloCat** — the leading mobile/WebGL URP shader developer
- **Unity Official docs** — Shader Performance, Mobile Optimization, WebGL Graphics
- **ARM/Mali**, **Meta/Adreno** — GPU vendor optimization guides
- **40+ GitHub repos** — NVJOB water shaders, FastPostProcessing, URP templates, and more

Full source list: see the `SOURCES` section at the bottom of each skill file.

## Contributing

PRs welcome. When adding or updating a skill:

1. Ground every rule in a verified source (link it)
2. Include working code snippets where possible
3. Test code templates compile against URP 14+ / Unity 2022.3+
4. Keep mobile/WebGL as the primary target

## License

MIT — see [LICENSE](LICENSE).
