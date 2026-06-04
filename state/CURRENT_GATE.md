# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Get R2 actually WORKING (uploads persist as r2://). R2 config is present but verification FAILED: post-redeploy uploads still land file://. Rob to fix the R2 secret value(s); executor re-verifies read-only. STOPPED.

## Done / verified (read-only)
- STT bridge btg-stt READY; WS routing confirmed.
- Five R2 names present (total 25 secrets); backend redeployed rev 00024; /api/health 200.

## R2 VERIFICATION FAILED (RESULTS 00:42Z)
- Newest uploads AFTER rev-00024 still file://:
  - video ad504088 (00:38:55): storage_uri=file://, status=failed, new error "ffmpeg error: ffmpeg version 5.1.9..." (secondary/downstream).
  - file 15c8350b (00:37:44): completed, file://.
- => R2 not persisting. _r2_enabled() effectively False OR R2 write fails -> file:// fallback. Config present != R2 working (as cautioned).
- Backend logs not retrievable this session (cerebrium logs flaky) to read the exact R2 error.

## Next step — ROB-ONLY: fix the R2 secret value(s) (executor does NOT read values)
1. Rob checks the R2 values locally (never in chat):
   - r2_endpoint well-formed: https://<accountid>.r2.cloudflarestorage.com (https, no trailing slash, NO bucket in path). Prior STATE.md:436 flagged this endpoint malformed — most likely culprit.
   - r2_enabled value is exactly the string true (lowercase) so the bool parses truthy.
   - r2_access_key / r2_secret_key = a valid R2 API token pair for r2_bucket.
2. Optionally: Rob reads backend App Logs (dashboard) for the 00:38 upload to see the exact object_store R2-fallback error.
3. Rob re-sets the corrected value(s), redeploy (confirm-before-live), re-test ONE upload.
4. (AUTO-RUN) executor re-checks newest source storage_uri scheme read-only. PASS only when r2://.

## Secondary (after R2 fixed)
- video ad504088 ffmpeg error — investigate separately once R2 works.

## After R2 verified r2://
- Re-upload ellis videos / use a good upload -> reprocess [ROB-ONLY] -> executor verifies STT end-to-end (transcribed, segments>0, /stream logs). Then TTS (ROB-ONLY spend).

## ROB-ONLY (carried)
- Fix/set R2 values; read backend logs (dashboard); upload/reprocess; redeploy (confirm-before-live); TTS/face. No secret values read/set by executor.

## Hard constraints
No secret values read/set by executor. No upload/authed-call/reprocess by executor. No redeploy without confirm-before-live. No blind retry of file:// ellis sources. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
