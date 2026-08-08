# Installing the `rust-performance` Skill

`SKILL.md` in this directory is a Claude Code Agent Skill distilled from *The
Rust Performance Book* by Nicholas Nethercote
(<https://nnethercote.github.io/perf-book/>, dual MIT/Apache-2.0). It is not
part of the book; it is a derived reference for use with an AI coding agent.

Claude loads a skill only when the task matches its `description`, so an
installed skill costs nothing until a Rust performance question comes up.

## Requirements

The file must live at `<skills-dir>/<skill-name>/SKILL.md`. Two rules matter:

- The containing **directory name** must match the `name:` field in the
  frontmatter — here, `rust-performance`.
- The file must be named `SKILL.md` exactly.

Copying `SKILL.md` to a bare path such as `~/.claude/skills/SKILL.md` will not
work.

## Option 1 — Install for every project (recommended)

Personal skills live in `~/.claude/skills/` and are available in every
directory you run Claude Code from.

```bash
mkdir -p ~/.claude/skills/rust-performance
cp SKILL.md ~/.claude/skills/rust-performance/SKILL.md
```

On Windows, the path is `%USERPROFILE%\.claude\skills\rust-performance\`.

## Option 2 — Install for one project

Project skills live in `.claude/skills/` inside the repository, and apply only
when Claude Code is run from that repository. Commit the directory and every
contributor on the team gets the skill.

```bash
mkdir -p /path/to/project/.claude/skills/rust-performance
cp SKILL.md /path/to/project/.claude/skills/rust-performance/SKILL.md
```

If you would rather not commit it, add `.claude/skills/` to the project's
`.gitignore` (or to `~/.config/git/ignore` for a global rule).

## Verifying the install

Start (or restart) Claude Code and run:

```
/skills
```

`rust-performance` should be listed. If it is not, check the directory name,
the filename, and that the frontmatter block is intact at the top of the file
(`---`, `name:`, `description:`, `---`).

To force it on demand rather than waiting for automatic matching:

```
/rust-performance
```

## What triggers it

Claude reads the `description` field to decide relevance. It matches prompts
about Rust runtime speed, memory usage, binary size, and compile times —
profiling, benchmarking, Cargo build configuration (`opt-level`, LTO,
`codegen-units`, allocators, linkers), heap allocations, type sizes, hashing,
iterators, I/O, bounds checks, and slow builds.

To broaden or narrow that, edit the `description` line in `SKILL.md`. It is the
only thing Claude sees before deciding to load the rest of the file, so
concrete triggering phrases work better than abstract summaries.

## Updating

The book is actively maintained; this skill is a snapshot. To refresh it, pull
the latest `perf-book`, re-derive the file, and copy it over the installed
version. Nothing caches it — the next session reads the new file.

## Note on provenance

The upstream book states that it contains no generative-AI material and accepts
none. Keep derived files like this one out of any pull request to
`nnethercote/perf-book`; the appropriate place for them is your own tooling
directories.
