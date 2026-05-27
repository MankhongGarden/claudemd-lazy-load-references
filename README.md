# Stop your CLAUDE.md from drowning out the working session

If you're running Claude Code seriously enough that your `CLAUDE.md` has grown past the 40K-character warning, you've already noticed: the agent loads the whole thing on every session start. Power-user configs hit 10-15K tokens of always-on context [before the user types anything](https://github.com/anthropics/claude-code/issues/33464). That's tokens taken from working memory, from MCP tool descriptions, from the file you're actually editing.

This repo describes a pattern I've been running on my own setup that defers most of the file to **on-demand reads**, keeping the boot context short while preserving every piece of guidance. Plus a Windows / PowerShell workaround for the safety block that prevents Claude Code from editing its own `CLAUDE.md` (it's classifier-blocked for good reasons — but you the user can sidestep it cleanly with a `!` prefix script).

> **Audience:** Claude Code power users running `CLAUDE.md` larger than ~30 KB, especially those with multi-project / multi-context setups where the file grew organically across sessions.

---

## The problem in one diagram

```
SESSION START                          ←── 1 MB / 200K tokens budget
├── system prompt (Anthropic)          ~3K tokens
├── MCP tool descriptions              ~15-30K tokens (depending on servers)
├── CLAUDE.md (user global)            ─── grows ─── 10-15K tokens
├── CLAUDE.md (project local)          ─── grows ─── 5-10K tokens
├── MEMORY.md                          ~1-3K tokens
├── ...other auto-loaded               ~2-5K tokens
└── working context                    ←── what's actually left
```

In my own setup mid-2026-05, the global `CLAUDE.md` had grown to 42 KB across:
- Windows debugging cookbook entries (Task Scheduler · GCM · WT defaults · TPM · ConPTY · ...)
- Multi-project boundary rules
- Cross-session state-sharing protocols
- Vendor-specific patterns (Stripe TH · Omise · LINE OA · LINE rich menu · ...)
- Classifier guardrail rules learned the hard way
- Browser-driving tool protocols (chrome-mcp · chrome-devtools)

Every section was useful **sometimes**. None was useful **every session**. But all of it was loaded into every session because that's how `CLAUDE.md` works.

## The pattern · lazy-load references

The fix is structural, not algorithmic. Split `CLAUDE.md` into two layers:

**Layer 1 · always loaded (the trimmed `CLAUDE.md`)**
- Identity / scope rules (who is this AI, what project space)
- Anti-pattern rules that fire often (classifier guardrails · scope creep tripwires)
- Pointers to layer 2 by symptom

**Layer 2 · loaded on demand (`references/<topic>.md`)**
- Bug recipes (Windows-specific gotchas · vendor-specific traps)
- Long protocol documents (multi-project boundary protocol · cross-session sync)
- Reference tables that don't need to live in the prompt

The trimmed `CLAUDE.md` includes one-line pointers:

```markdown
**เจอ symptom Windows ที่ไม่อยู่ในนี้** (Task Scheduler hang · GCM popup spam · 
PAT recovery · WT default terminal trap · ...) → อ่าน 
`D:\ClaudeData\.claude-personal\references\windows-debugging-cookbook.md`
```

When the agent hits a symptom that matches the pointer, it Reads that reference file then continues. If the symptom doesn't match anything in the pointers, the agent never opens those files — context stays clean.

## What "lazy-load reference" looks like in practice

Three real examples from my setup:

**Example 1 · Windows debugging cookbook**

The cookbook has ~30 specific bug recipes — each a "symptom → root cause → fix" entry for things like:
- Task Scheduler MSI install hang
- GCM popup spam after PAT rotation
- Windows Terminal hot-reload not respecting settings.json
- TPM fTPM stuck flag

These hit maybe 1-2 times per month. Pre-trim, ~12 KB of context every session. Post-trim, 0 KB unless the agent hits one of those symptoms — then it reads `references/windows-debugging-cookbook.md` once for that session.

**Example 2 · Project-specific brand policy**

The brand policy doc was ~6 KB and only relevant when writing public-facing marketing copy. Pre-trim, loaded every session including pure backend / DB work where it was irrelevant. Post-trim, lives in `references/brand-policy.md`, referenced from the trimmed `CLAUDE.md` only when working in `marketing/` or running a publish skill.

**Example 3 · Vendor cookbook**

Each vendor (Stripe TH · Omise · LINE OA · ...) had its own block of gotchas. Pre-trim, all loaded together (~8 KB). Post-trim, split into `references/stripe-th.md` · `references/omise.md` · etc. with one-line pointers in `CLAUDE.md`. Now only the vendor relevant to the current task gets loaded.

## The Windows / classifier workaround

Here's the wrinkle if you want the agent itself to perform the extraction: Claude Code's classifier blocks the agent from editing its own `CLAUDE.md` (treated as "Self-Modification" of agent-loaded config — see the [Anthropic security model](https://docs.claude.com/en/docs/claude-code/security)). User authorization can't override this block. So if you ask the agent to "go ahead and trim my CLAUDE.md from 42K to 18K", it'll refuse.

The clean workaround: have the agent **draft** the new content + write a `.ps1` script that applies the changes atomically with a backup. Then **you** run the script via the `!` prefix in your prompt (Claude Code's `!` runs commands under the user's identity, not the agent's — classifier scope doesn't apply).

Skeleton:

```powershell
# scripts/trim-claudemd.ps1 · generated by the agent, run by you with `!`
$src = "$env:CLAUDE_CONFIG_DIR\CLAUDE.md"
$bak = "$src.bak.$(Get-Date -Format yyyyMMddHHmmss)"

# 1. Atomic backup
Copy-Item $src $bak -Force
Write-Host "backup: $bak"

# 2. Write the trimmed CLAUDE.md (UTF-8 no BOM)
$trimmed = @'
... trimmed content here, drafted by the agent ...
'@
[System.IO.File]::WriteAllText($src, $trimmed, [System.Text.UTF8Encoding]::new($false))
Write-Host "wrote: $src ($((Get-Item $src).Length) bytes)"

# 3. Write the reference files
foreach ($pair in @(
  @{ Name = "windows-debugging-cookbook.md"; Body = @'
... cookbook content ...
'@ },
  @{ Name = "brand-policy.md"; Body = @'
... brand policy content ...
'@ }
)) {
  $ref = "$env:CLAUDE_CONFIG_DIR\references\$($pair.Name)"
  New-Item -ItemType Directory -Force -Path (Split-Path $ref) | Out-Null
  [System.IO.File]::WriteAllText($ref, $pair.Body, [System.Text.UTF8Encoding]::new($false))
  Write-Host "wrote: $ref ($((Get-Item $ref).Length) bytes)"
}

Write-Host "done"
```

You invoke it via:

```
!powershell -ExecutionPolicy Bypass -File "D:\path\to\trim-claudemd.ps1"
```

The `!` prefix runs the command under your shell, not the agent's. Output flows back into the conversation. Classifier never enters the picture — you're the actor, the agent is the scribe.

If anything goes wrong, the backup is at `CLAUDE.md.bak.<timestamp>` — restore with a Copy-Item.

## Where the symptom pointers go

Inside the trimmed `CLAUDE.md`, the pointer syntax I use is:

```markdown
**เจอ symptom เฉพาะ Windows ที่ไม่อยู่ในนี้** (Task Scheduler hang · GCM popup spam · 
PAT recovery · WT default terminal trap · WT settings hot-reload · AMD fTPM stuck flag · 
ConPTY headless · Get-PfxCertificate hang · Schannel TLS cert recovery · hook visible-flash · 
`claude mcp add` flag bug · `claude -p` slow startup · `anthropics/claude-code` issue bot 
duplicate flag · Task Scheduler MSI PS7 install) → อ่าน 
`D:\ClaudeData\.claude-personal\references\windows-debugging-cookbook.md`
```

The parenthetical lists the **symptom keywords** that should fire the Read. The agent matches against task wording / error text and decides "do I need this reference?" before reading. Most sessions don't trigger any reference Read — the pointers are just there, costing maybe 200-300 tokens total versus 12 KB if the full content lived in the trimmed file.

For multilingual setups (mine is Thai + English), pointers can be in your working language without affecting agent comprehension — the agent reads both.

## What I cut · the surprising tally

Starting from 42 KB / ~10.5K tokens, the trim split into:

| Layer | Before | After | Δ |
|---|---|---|---|
| Always-loaded `CLAUDE.md` | 42 KB / ~10.5K tok | 18 KB / ~4.5K tok | -57% |
| `references/windows-debugging-cookbook.md` | (inline) | 12 KB / ~3K tok | n/a (lazy) |
| `references/cross-session-state.md` | (inline) | 5 KB / ~1.3K tok | n/a (lazy) |
| `references/brand-policy.md` | (inline) | 6 KB / ~1.5K tok | n/a (lazy) |
| `references/vendor-cookbook/*.md` | (inline) | 8 KB / ~2K tok | n/a (lazy) |

Total content actually grew by ~30% post-trim (because I expanded some sections that I'd been keeping artificially short to save context budget) — but the always-loaded portion dropped 57%. Working memory recovered ~6K tokens per session.

## When NOT to use this pattern

- Your `CLAUDE.md` is under 5 KB. The savings don't justify the indirection.
- You only use Claude Code for one project. Everything is relevant every session; lazy-loading just costs you Reads.
- The content is genuinely cross-cutting (identity · scope · anti-patterns that fire on every task). That stays in layer 1.

The litmus test for each section: *"in the last 10 sessions, how often was this guidance actually used?"* If the answer is "0-2 times", it's a candidate for layer 2.

## Related upstream issues

- [`anthropics/claude-code#33464`](https://github.com/anthropics/claude-code/issues/33464) — Feature request: native token compression for `CLAUDE.md` (open). The OP runs 27 agent files and built their own algorithmic token-trim tool, and lists "split files and load on-demand (requires building your own cold-start architecture)" as one of today's three options. This guide is that third option, written down. The request proposes two shapes — automatic compression and *semantic/progressive loading* — the latter being the native version of what this pattern does by hand.
- [`anthropics/claude-code#40895`](https://github.com/anthropics/claude-code/issues/40895) — A Max-plan user asking why 2 Opus prompts ate 20% of a 5-hour window, with MCP servers + skills + `CLAUDE.md` all loaded. Closed as stale, but the question — "how much does my loaded context cost me?" — is exactly the pain this pattern targets.

One caution, because it keeps the pattern honest: loaded-context size is *frequently blamed* for rate-limit exhaustion but isn't always the cause. In [`#41424`](https://github.com/anthropics/claude-code/issues/41424) a user whose `CLAUDE.md` was only 3% of context still hit limits immediately — their own `/context` traced it to a browser tool's cumulative token attribution, not their instruction files. Trimming `CLAUDE.md` recovers working-memory budget and lowers per-prompt context cost; it is not a fix for metering anomalies. Use it for what it does.

If Anthropic ships native progressive loading (#33464 proposes one shape · `skillOverrides` config already exists for skills), this manual pattern becomes redundant. Until then, it works on stock Claude Code with no plugin / no settings change required.

## Sponsors

This guide is free. If it freed up enough context that your Opus session stops hitting the 5-hour rate limit at prompt 12:

- ⭐ Star this repo so other power users find it (GitHub ranks by stars)
- 💛 [GitHub Sponsors](https://github.com/sponsors/MankhongGarden) — sustains weekend writeups like this

## License

MIT — see [LICENSE](LICENSE). Use anything here however you want.
