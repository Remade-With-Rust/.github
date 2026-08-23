# Skills

Agent skills for building on the Remade With Rust stack. Drop one into your agent's skills
directory and it loads automatically when the work matches.

| Skill | What it's for |
|---|---|
| [`building-the-new-internet`](building-the-new-internet/) | Starting and shipping a Rust app, service or website that is ready to deploy onto the MATA distributed cloud the day it goes live. |

## building-the-new-internet

The single playbook for the whole stack. It answers **"what do we use for X"** — mID for
identity, SpaceDB for storage, remade_ffmpeg_rs for media, FFAI for AI, rusty_alloc for
allocation, rusty_zstd for compression, thoth for UI chrome, Deputy for the supply chain — and
then the parts that decide whether your app can actually run on a mesh: API-first architecture,
the per-entry CRDT law, the five deploy seams, the performance and measurement discipline, and
the build/validate workflow.

```
building-the-new-internet/
├── SKILL.md          the stack table, the day-one scaffold, the pre-flight checklist
├── stack.md          every dependency decision, with versions, features and traps
├── architecture.md   general primitives, API-first, per-entry CRDT, the trust boundary
├── deploy.md         the road to the distributed cloud, and what is actually live today
├── performance.md    the five moves, the measurement bar, when unsafe is earned
├── workflow.md       compile gates, isolation harnesses, the curiosity discipline
└── ui.md             Dioxus footguns and the glyph/token/a11y crates
```

### Install

**Claude Code** — copy the directory into your skills folder:

```sh
git clone https://github.com/Remade-With-Rust/.github rwr-meta
cp -r rwr-meta/skills/building-the-new-internet ~/.claude/skills/
```

Per-project instead of global: copy it to `.claude/skills/` in the repo.

**Any other agent** — the files are plain Markdown with YAML frontmatter. Point your tool at
`SKILL.md`; it links the six reference files by name.

### Reading it as a human

You don't need an agent. Start at `SKILL.md` — the stack table and the one-line test are the
whole contract in two screens — then read whichever reference file your task touches.

### Keeping it honest

Every status and version in these files is what was true when written, including the
unflattering ones: which services are Preview rather than GA, which crates are not on crates.io
yet, and where a C dependency still enters the tree. If you find one that has drifted, a PR
correcting it is the most useful contribution you can make here.

## License

MIT.
