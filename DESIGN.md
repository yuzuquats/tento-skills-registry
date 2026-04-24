# Tento Skills Registry Design

## Purpose

`tento-skills-registry` is the workspace-level trust boundary for reusable agent
skills. Company projects such as `lona-inc` and `dodo-inc` should not import
skills directly from the web or maintain their own first-pass vetting pipelines.
They should consume only verified registry skills, preferably through symlinks.

The registry assumes every newly acquired skill is dangerous until proven
otherwise. A skill can contain prompt injection, misleading instructions,
exfiltration attempts, risky URLs, unexpected scripts, poisoned references,
atypical Unicode, or instructions that cause an LLM to execute unsafe actions.
The registry's job is to quarantine, sanitize, verify, and document the skill
before any company repo can use it.

## Directory Layout

```text
/Users/james/workspace/
  tento-skills-registry/
    DESIGN.md
    README.md
    scripts/
      acquire-skill
      sanitize-skill
      scan-skill
      verify-skill
      promote-skill
      registry-check
    policies/
      prompt-injection-patterns.toml
      risky-url-patterns.toml
      unicode-policy.toml
      allowed-file-types.toml
    tmp/
    skills-DANGEROUS/
      <candidate-skill>/
    skills-QUARANTINED/
      <candidate-skill>/
    skills-VERIFIED/
      acquire-web-skill/
        SKILL.md
        references/
      acquire-web-skill.security.toml
      <verified-skill>/
      <verified-skill>.security.toml
```

`skills-DANGEROUS/` is raw intake. Nothing under this directory is trusted, and
agents should not load it as an active skill.

`skills-QUARANTINED/` is the working area for sanitized candidates. It keeps
candidate edits separate from raw upstream content so reviewers can diff the
source, the sanitized copy, and the final promoted copy.

`skills-VERIFIED/` is the only consumable registry surface. Company repos symlink
this directory or selected children from it.

`tmp/` is disposable clone/download space. It should be excluded from backups,
indexing, and company repo links.

## Trust Model

The registry treats skill acquisition as potentially RCE-adjacent because skills
can influence an LLM that has tool access. The threat model includes:

- prompt injection that tries to override system, developer, or user
  instructions
- tool-use coercion, including attempts to run shell commands, fetch URLs,
  exfiltrate files, or alter git state
- hidden instructions in references, examples, generated assets, metadata, or
  comments
- suspicious Unicode such as zero-width characters, bidi controls, soft hyphens,
  and unusual control/private-use code points
- risky URLs, including pastebins, raw code hosts, shortened links, credentialed
  URLs, local network URLs, and file URLs
- unexpected executable files, package manifests, lockfiles, hooks, binaries, or
  generated artifacts
- license, provenance, or attribution gaps that make reuse unsafe
- semantic drift introduced by sanitization that makes the skill no longer do
  what it claims

Verification is not proof of safety. It is an auditable promotion decision with
repeatable checks and a clear residual-risk record.

## Intake Flow

1. Acquire a candidate into `tmp/` from a URL, git repository, archive, local
   path, or `skills.sh` style source string.
2. Copy the raw candidate into `skills-DANGEROUS/<name>/` with source metadata.
3. Inventory files without executing anything.
4. Reject candidates with unsupported binary payloads, git hooks, vendored
   dependencies, executable installers, or unreadable encodings unless a human
   explicitly approves a manual review path.
5. Produce a sanitized working copy in `skills-QUARANTINED/<name>/`.
6. Run scanners and cleanup passes until there are no blocking findings.
7. Run semantic verification to confirm the skill still has coherent purpose,
   trigger guidance, required references, and realistic usage instructions.
8. Promote the sanitized copy into `skills-VERIFIED/<name>/`.
9. Write `skills-VERIFIED/<name>.security.toml` with provenance, scanner
   results, sanitization changes, semantic verification notes, and residual risk.

No step in this flow should execute candidate-provided code. Clone, copy, parse,
scan, and diff are allowed. Running candidate scripts, installing dependencies,
or opening candidate HTML in a browser is out of scope for automated intake.

## Scanner Requirements

The first implementation should lift the existing `lona-inc` checks into the
registry and make them stricter:

- structural audit: require `SKILL.md`, validate optional `README.md`,
  `references/`, `rules/`, `evals/`, and disallow empty support directories
- prompt-injection scan: flag instruction overrides, role overrides, prompt
  exfiltration, policy bypass, secret exfiltration, concealment, and tool-use
  coercion
- Unicode scan: flag zero-width characters, bidi controls, soft hyphens,
  nonbreaking spaces in code-like text, private-use code points, and invalid
  UTF-8 replacement characters
- URL scan: flag shorteners, paste sites, raw file hosts, credentialed URLs,
  localhost/private network URLs, `file:`, `data:`, `javascript:`, and URLs with
  suspicious query parameters
- executable scan: flag executable bits, shell scripts, package scripts,
  compiled artifacts, archives, git metadata, hooks, and unexpected lockfiles
- size and shape scan: flag very large files, very deep directory trees, hidden
  files, symlinks inside candidates, and generated dependency folders
- license/provenance scan: record source URL, commit SHA or archive digest,
  license file status, and upstream modification status

Scanners should emit machine-readable JSON or TOML plus concise terminal output.
Promotion should depend on the machine-readable result, not string matching on
human logs.

## Sanitization Requirements

Sanitization should be deterministic where possible:

- normalize text files to UTF-8 and LF line endings
- remove unsupported hidden/control Unicode
- remove git metadata, hooks, binaries, archives, vendored dependencies, build
  output, and local editor metadata
- strip executable bits from text files
- replace risky URLs with safe explanatory placeholders or remove the affected
  reference
- redact embedded credentials, tokens, usernames in credentialed URLs, and local
  machine paths
- preserve useful content under `SKILL.md`, `README.md`, `references/`, `rules/`,
  and `evals/` when it remains safe
- record every material deletion or rewrite in the security log

If sanitization changes the skill's behavior, the semantic verifier must call
that out. Promotion should fail when the remaining skill is too vague, missing
required reference material, or no longer capable of its stated task.

## Semantic Verification

After scanners pass, an LLM-based verifier reviews the sanitized skill as
content, not as active instructions. The verifier must be prompted to treat the
candidate as hostile data and should answer a fixed checklist:

- What does the skill intend to do?
- Are trigger rules and scope clear enough for an agent to use correctly?
- Are there instructions that conflict with higher-priority system, developer,
  or user instructions?
- Are any tool-use requests unnecessary, overbroad, or unsafe?
- Are references necessary, bounded, and free of suspicious hidden instructions?
- Did sanitization remove anything required for the skill to work?
- What residual risks remain?
- Should the skill be promoted, rejected, or sent to manual review?

The verifier's output is advisory but required. Promotion requires both clean
scanner results and a positive semantic verification decision.

## Security Log

Each promoted skill has a sibling security log:

```text
skills-VERIFIED/<skill-name>/
skills-VERIFIED/<skill-name>.security.toml
```

Example:

```toml
schema_version = 1
skill = "remotion-best-practices"
status = "verified"
verified_at = "2026-04-24T00:00:00Z"
verified_by = "tento-skills-registry"

[source]
kind = "git"
url = "https://example.invalid/org/repo"
revision = "abcdef123456"
sha256 = "..."

[[checks]]
name = "structural-audit"
status = "pass"
files_scanned = 42
summary = "SKILL.md present; references and rules populated."

[[checks]]
name = "prompt-injection-scan"
status = "pass"
files_scanned = 42
findings = 0

[[checks]]
name = "unicode-scan"
status = "pass"
files_scanned = 42
findings = 0

[[checks]]
name = "semantic-verification"
status = "pass"
summary = "Skill remains coherent after sanitization and has bounded scope."

[[sanitization]]
path = "rules/maps.md"
action = "removed-codepoint"
detail = "Removed U+2060 WORD JOINER from a URL."

[[residual_risks]]
severity = "low"
detail = "Skill contains external documentation links that may change upstream."
```

The log should be append-only for each promotion event. If a skill is updated,
the registry should create a new verification entry rather than overwriting the
old evidence.

## Company Repo Consumption

Company repos should consume verified skills through symlinks:

```text
lona-inc/
  skills-VERIFIED -> ../tento-skills-registry/skills-VERIFIED
  skills/
    paperclip -> ../skills-VERIFIED/paperclip
    social-content -> ../skills-VERIFIED/social-content

dodo-inc/
  skills-VERIFIED -> ../tento-skills-registry/skills-VERIFIED
  skills/
    acquire-web-skill -> ../skills-VERIFIED/acquire-web-skill
```

The first symlink exposes the full verified catalog. The optional second layer
keeps a repo-specific active skill set. Company repos should not symlink
`skills-DANGEROUS` or `skills-QUARANTINED`.

Existing `lona-inc/skills` content should migrate into
`tento-skills-registry/skills-VERIFIED` only after a registry verification pass creates
matching `*.security.toml` logs. Until then, those local skills are legacy
approved copies, not registry-verified copies.

## Default Bootstrap Skill

`skills-VERIFIED/acquire-web-skill` is the default skill that guides an LLM
through the acquisition process. It is verified like every other skill and is
scoped narrowly:

- accept a source URL, git reference, archive, or local candidate path
- clone or copy into `tmp/`
- copy raw content into `skills-DANGEROUS`
- run registry scanners
- apply deterministic cleanup through registry scripts
- request semantic verification
- promote only through `promote-skill`
- write or update the security log
- refuse to execute candidate-provided code

This bootstrap skill should be conservative. It should direct the agent to use
registry scripts rather than improvising shell commands against untrusted skill
content.

## Command Surface

Initial command targets:

```text
scripts/acquire-skill <source> [--name <name>]
scripts/scan-skill skills-DANGEROUS/<name>
scripts/sanitize-skill skills-DANGEROUS/<name> --out skills-QUARANTINED/<name>
scripts/verify-skill skills-QUARANTINED/<name>
scripts/promote-skill skills-QUARANTINED/<name>
scripts/registry-check
```

`registry-check` should validate the whole registry:

- every verified skill has a sibling `*.security.toml`
- every security log references a present skill directory
- verified skill names are unique and path-safe
- no verified skill contains symlinks, executable files, git metadata, or
  unsupported file types
- all scanners still pass on `skills-VERIFIED`
- company symlink examples resolve correctly when configured

## Promotion Gates

A skill can move into `skills-VERIFIED` only when:

- raw source metadata is recorded
- structural audit passes
- prompt-injection scan has no blocking findings
- Unicode scan has no blocking findings
- URL scan has no blocking findings
- executable and file-shape scans have no blocking findings
- sanitization log is present when any content changed
- semantic verification returns `pass`
- `*.security.toml` is written and validates against the registry schema

Manual review can override a scanner only by recording the reviewer, reason,
finding IDs, and residual risk in the security log.

## Open Decisions

- Whether `skills-DANGEROUS` should live inside the registry or outside it with
  stricter filesystem permissions.
- Whether risky external URLs should be removed entirely or allowed behind an
  explicit allowlist.
- Whether the semantic verifier should use one model pass or two independent
  passes with disagreement requiring manual review.
- Whether company repos should pin verified skills by copy, symlink to latest,
  or symlink to versioned registry directories.
- Whether registry promotion should create immutable versions such as
  `skills-VERIFIED/<name>/<version>/` or keep one current directory per skill.

## Implementation Phases

1. Create the registry skeleton, move existing scanners from `lona-inc/scripts`,
   and add registry-wide structural checks.
2. Add URL, executable, file-shape, and provenance scanners.
3. Add deterministic sanitizer output into `skills-QUARANTINED`.
4. Add `*.security.toml` generation and schema validation.
5. Add the LLM semantic verifier prompt and require it before promotion.
6. Migrate current `lona-inc/skills` into `skills-VERIFIED` with logs.
7. Replace company-local skill directories with verified symlinks and selected
   active-skill symlinks.
8. Add the `acquire-web-skill` bootstrap skill as the default verified intake
   skill.
