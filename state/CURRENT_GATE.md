# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Enable R2 on the backend so uploads persist (r2://), as prerequisite to STT validation. HALTED: the five R2 secrets are reported set by Rob but are NOT present in the Cerebrium secret store the backend reads. Resolve the secret-store discrepancy before redeploy.

## HALT — precondition not met (RESULTS 23:20Z)
- `cerebrium secrets list` (the CLI used all session, project p-a907d7c5) shows 20 secrets, none of r2_enabled/r2_endpoint/r2_access_key/r2_secret_key/r2_bucket. Count unchanged.
- Did NOT redeploy: with R2 absent, _r2_enabled() is still False; a redeploy would change nothing and waste a confirm-before-live cycle.

## Done / verified (unchanged)
- STT bridge btg-stt READY (/healthz 200). Backend rev 00023 wired, /api/health 200, WS routing to /stream confirmed (101).
- Ellis 5 video sources are file:// and unretrievable; failed upstream of STT. R2 is the prerequisite to fix this.

## Next step — ROB-ONLY: resolve where the R2 secrets actually went
1. (Rob) Re-check how/where the five R2 names were set:
   - Was it CLI or dashboard? If CLI, exact command + did it return success or error? (Earlier `cerebrium secret set NAME VALUE` worked; `secrets add name="value"` may be a version mismatch.)
   - Dashboard: "Project Secrets" vs an app "Secrets" tab — backend reads Cerebrium PROJECT secrets for p-a907d7c5.
   - Is Rob's CLI pointed at the same project as the executor's? Rob's `cerebrium secrets list` should match the 20 the executor sees; if Rob sees the r2_ names and executor does not, the two target different projects.
   - Confirm names locally without values: `cerebrium secrets list | grep -iE 'r2_'`.
2. Once the five r2_ names appear in `cerebrium secrets list` for p-a907d7c5 (executor re-verifies name-only), proceed:
   - (CHECKPOINT/confirm-before-live) backend redeploy (deploy spec in gate/policy).
   - (AUTO-RUN) /api/health smoke.
   - (AUTO-RUN, read-only) verify R2 active WITHOUT executor upload — see method note below.
   - (AUTO-RUN) RESULTS + gate update. Stop at next ROB-ONLY (the actual upload/reprocess).

## R2-active verification method (no executor upload)
Options to confirm R2 is live without the executor performing an upload:
- Read the backend's own /api/health or a debug/status route if it reports r2_enabled (check routes read-only).
- Inspect upload._RECENT_UPLOAD_EVENTS / a status endpoint that exposes storage_uri scheme of the next Rob-performed upload.
- Simplest: Rob does ONE small fresh upload (ROB-ONLY) and executor reads the resulting storage_uri scheme (r2:// vs file://) read-only. (Upload stays Rob's action.)

## After R2 verified + a durable source exists
- Re-upload ellis video (or test persona) [ROB-ONLY] -> reprocess [ROB-ONLY] -> executor verifies STT end-to-end (transcribed, segments>0, /stream logs).
- Then TTS bridge (ROB-ONLY spend).

## ROB-ONLY (carried)
- R2 secret values + correct store/project.
- Upload/re-upload; reprocess trigger; backend redeploy (confirm-before-live); TTS/face GPU.
- No secret values set by executor.

## Hard constraints
No secret values set by executor. No prod DB writes/uploads by executor. No backend redeploy without confirm-before-live AND precondition met. No blind retry of file:// ellis sources. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
