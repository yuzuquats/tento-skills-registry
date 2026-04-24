# Tento Skills Registry

Workspace-level registry for untrusted skill intake and verified skill
consumption.

## Trust Boundaries

- `skills-DANGEROUS/`: raw candidate intake. Do not load these as active skills.
- `skills-QUARANTINED/`: sanitized candidates under review.
- `skills-VERIFIED/`: the only directory company repos should symlink from.

## Current Commands

```bash
scripts/scan-skill <skill-dir> [<skill-dir> ...]
scripts/registry-check
```

`scan-skill` runs the structural audit, prompt-injection scanner, Unicode risk
scanner, risky URL scanner, and file-shape scanner. `registry-check` validates
every current verified skill and checks that each has a sibling
`*.security.toml` log.

The current implementation is phase one of `DESIGN.md`; deterministic
sanitization, semantic verification automation, promotion, and acquisition are
still pending.
# tento-skills-registry
