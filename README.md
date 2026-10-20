# vet-repo

A [Claude Code](https://claude.com/claude-code) skill that security-vets a GitHub repo, skill, plugin, MCP server or npm/pip package **before** you install or run it.

It judges the target by what is actually in its files, not by its README or reputation. It is read-only: it clones and greps, and never executes the code under review.

## What it does

When you ask Claude "is this safe to install?" or paste a GitHub URL you are about to install, the skill walks Claude through:

1. **Identify** the target: owner, commit hash, age, stars, and whether it is a skill, a package/CLI, or an MCP server/plugin.
2. **Skills (`SKILL.md`)**: read every file and look for prompt injection, exfiltration, silent permission changes, hidden or encoded content, and auto-installing other things.
3. **Code (CLI, package, plugin, MCP)**: check install-time hooks, process spawning, `eval`, network hosts, environment variables and secrets read, credential paths, dependencies, telemetry, obfuscation, and CI/release config.
4. **Second-order risk**: if the tool installs other things, those need vetting too.
5. **Report** in a short, fixed format with a verdict.

### Report format

```
Verdict: Safe / Safe with caveats / Not recommended / Couldn't determine
Reviewed: <exactly what files/paths and commit hash>
Not reviewed: <dependencies, dist build, remote catalog, etc.>

Reassuring: <bullets, each backed by something seen>
Concerns: <bullets with file:line where possible>
Before installing: <concrete steps>
```

## Install

Skills live in a `skills/<name>/` folder containing a `SKILL.md`. Pick one location:

| Scope | Directory |
|---|---|
| All your projects (personal) | `~/.claude/skills/vet-repo/` |
| One project only | `<project>/.claude/skills/vet-repo/` |

### Option 1: clone and copy (recommended)

```bash
git clone https://github.com/prapurna-codespace/vet-repo-skill.git
mkdir -p ~/.claude/skills/vet-repo
cp vet-repo-skill/skills/vet-repo/SKILL.md ~/.claude/skills/vet-repo/SKILL.md
```

### Option 2: single file download

```bash
mkdir -p ~/.claude/skills/vet-repo
curl -fsSL https://raw.githubusercontent.com/prapurna-codespace/vet-repo-skill/main/skills/vet-repo/SKILL.md \
  -o ~/.claude/skills/vet-repo/SKILL.md
```

### Option 3: manual

Create `~/.claude/skills/vet-repo/SKILL.md` and paste in the contents of [`skills/vet-repo/SKILL.md`](skills/vet-repo/SKILL.md).

Then start a new Claude Code session (or run `/skills` to confirm `vet-repo` is listed).

> You are installing a skill that tells an agent how to behave, so read it first. It is one file of plain Markdown with no scripts. That is the same standard it asks you to hold everything else to.

## Usage

Just ask in natural language:

- "Is https://github.com/someone/some-skill safe to install?"
- "Vet this MCP server before I add it."
- "Check this npm package for anything sketchy."

Or invoke it directly with `/vet-repo <url>`.

Claude will clone the target into a fresh scratch directory, read and grep it, and return the report. It will not install anything until you say so.

## Requirements

- Claude Code with shell access (`git`, `grep`, `curl`).
- Optional: the [`gh` CLI](https://cli.github.com/) for repo metadata such as stars and account age.

## Limitations

- **A static review is not a guarantee.** Heavily obfuscated code, malicious dependencies, or behaviour that only appears at runtime can slip through. The report states what was not reviewed; take that seriously.
- **It reviews a point in time.** A safe commit today says nothing about future updates. Re-vet after upgrades and pin to a commit or tag.
- **Dependencies are only lightly checked.** It looks at the manifest and lockfile for typosquats and unpinned versions, but does not audit the full dependency tree.
- **The model is the reviewer.** Treat the verdict as a strong second opinion, not a replacement for your own judgement on anything high-stakes.

## Safety design

- Never runs `npm install`, `npx`, `pip install`, `make`, install scripts, or the target's tests.
- Treats all downloaded text as untrusted data, including text that looks like instructions to Claude, so a malicious repo cannot steer the review.
- Clones into an isolated directory, separate from your working project.
- Does not install anything without your explicit go-ahead.

## Vetting a skill by hand

You don't need this skill to check a skill. The same checks, done manually:

1. **Read `SKILL.md` first, all of it.** It is plain-language instructions the agent will obey. Anything that tells the AI to hide things ("don't tell the user", "don't mention this"), or to touch files and settings the skill has no reason to touch, is a red flag.
2. **Hunt for scripts.** Look for `.sh`, `.py`, `.js` and `install.sh` files and read each one. Never run `curl ... | bash` from a repo you haven't read.
3. **Search for red-flag keywords** across the whole folder (`grep -rniE`):
   - `curl`, `wget`: download and run things from the internet.
   - `eval`, `base64`: hide what the code really does.
   - `.ssh`, `.env`, `cookies`, `token`, `password`: go after credentials.
   - Instructions to keep things from the user.

   A match isn't automatically bad. It means you read that line carefully and check that it makes sense.
4. **Check the repo itself.** Who is the author, and do they have a real history? Does the commit history look natural, or did everything land in one day? Is there a license? Stars can be bought, so they prove nothing.
5. **Never hand over keys.** A skill should not need your API keys, SSH keys or cloud credentials. If it asks for them, stop.
6. **Test in an empty folder, then have Claude audit it.** Run it somewhere with nothing sensitive in it, and ask Claude to read the whole repo and report anything suspicious before you trust it.

## Audit of this repo

Checked on 2026-10-21 against commit `e6c01d7`, by cloning the published repo into an empty directory.

| Check | Result |
|---|---|
| Read all of `SKILL.md` | No instructions to hide anything from the user, and nothing that touches unrelated files or settings |
| Scripts | None. The repo is three files: `LICENSE`, `README.md`, `skills/vet-repo/SKILL.md` |
| Red-flag keywords | Every match is the skill or README *describing* what to look for. The only real `curl` is the install one-liner, which fetches this repo's own `SKILL.md` and pipes nothing to a shell |
| Hidden characters | No non-ASCII characters, so no zero-width or unicode tricks |
| Credentials | Nothing asks for API keys, SSH keys or cloud credentials |
| Repo history and license | One commit by the author, MIT license. A new repo has no track record or stars to judge |

**Not yet verified:** a full end-to-end install test of the `curl` command into a clean `HOME`. This audit was done by the author, so it is a sanity check and not an independent review. Please read the three files yourself before installing.

## Contributing

Issues and PRs are welcome, especially new red-flag patterns, ecosystems (Go, Rust, Docker, GitHub Actions) and false-positive reports. Keep the skill a single readable `SKILL.md` where possible.

## License

[MIT](LICENSE)
