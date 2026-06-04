# NEXT REPLY

status: PENDING
updated_at: 2026-06-04T04:55:00Z
consumed_at:
consumed_by:
gate_commit: 5495e05
classification_hint: DEPLOY approved confirm-before-live backend redeploy

## body
Rob approves the confirm-before-live backend redeploy for Gate A.

Approval scope:
- Redeploy backend only to pick up the Qwen/provider-agnostic persona extraction fix.
- Command: cd backend && cerebrium deploy
- Use logged background deploy as before.
- Smoke: /api/health returns 200 after deploy.
- Rollback/stop: if deploy or smoke fails, stop and report exact error; do not reprocess.

After backend deploy + health smoke succeeds:
- Stop for Rob to trigger the authed reprocess of source 294c44ea-9784-42eb-988a-701a11d7c448.
- Executor verifies read-only after Rob triggers reprocess:
  - source completes, not failed
  - error_message empty
  - PersonaMemory rows are created for the persona
  - Qwen/provider-agnostic extraction path ran

Hard constraints:
- Do not trigger reprocess/upload yourself.
- Do not start TTS or face/avatar.
- Do not handle or print secrets.
- Keep Gate B, zero speaker segments / HF-token / pyannote, separate.
