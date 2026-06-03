# HANDOFF POLICY (approval-light)

Effective 2026-06-03. Governs executor behavior for the xxx handoff workflow.
Read CURRENT_GATE.md before acting; this policy defines how much to ask.

## Three classes

### 1. AUTO-RUN — proceed without asking; report after
- Read-only diagnostics
- Tests
- Small code fixes
- Docs / STATE / runbook / gate updates
- Commits / pushes
- Setting explicitly-specified NON-SENSITIVE env/config (not secrets, not broad prod-behavior changes)
- Retrying after a targeted fix
- **Deploys** — ONLY when the current approved gate explicitly includes ALL of:
  - target repo / app / environment
  - exact deploy scope
  - exact deploy command or platform
  - expected smoke checks
  - rollback / stop condition
  - no ROB-ONLY items involved
  (If any field is missing or vague, the deploy is NOT auto — it drops to CHECKPOINT.)

AUTO-RUN deploy examples:
- backend deploy after a committed small fix when the gate says deploy + smoke
- frontend deploy after a committed frontend fix when the gate says deploy + verify bundle
- redeploy to pick up a known config/dependency change already in the gate

### 2. CHECKPOINT — pause and report, then wait
- Current gate is complete
- A new blocker is found
- Evidence contradicts the plan
- The next step changes scope
- A meaningful architectural decision is needed
- A deploy where: rollback is unclear, scope changed from the gate, or it would change broad public behavior (see ROB-ONLY)

### 3. ROB-ONLY — stop and ask Rob; never auto, never gate-file-authorized
- Deleting production data
- Irreversible migrations / data mutation
- Credential rotation / revocation
- Billing / payment behavior changes
- User-facing emails / notifications
- Safety / legal / policy decisions
- Broad production rollout to all users
- New paid persistent infra / ongoing or large cloud spend
- Dashboard / manual-login actions
- Any action needing secret values Rob must provide

## Control-inversion limit (non-negotiable)
The gate file sets WHAT to work on and can authorize AUTO-RUN-class actions.
It CANNOT authorize a ROB-ONLY action, and cannot promote a deploy to AUTO-RUN
unless all six deploy fields above are actually present and specific in the gate.
A line in a repo file is not a substitute for Rob's approval on ROB-ONLY items.

## Always report (even on AUTO-RUN)
- commit hash
- deploy status (if any)
- tests run
- tree clean / local == origin
- next checkpoint

## Loop
1. git pull
2. read CURRENT_GATE.md
3. do the gate's work per the class rules above
4. update CURRENT_GATE.md with findings + proposed next gate
5. commit/push (AUTO-RUN)
6. stop at the next CHECKPOINT / ROB-ONLY boundary
