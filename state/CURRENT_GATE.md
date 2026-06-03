# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate STT end-to-end. R2 config now present and backend redeployed (rev 00024). Next: confirm R2 actually WORKS (write yields r2://), which needs a Rob upload — then re-upload + reprocess to exercise STT. STOPPED at the upload boundary.

## Done / verified (read-only)
- STT bridge btg-stt READY (/healthz 200). WS routing to /stream confirmed (101).
- Five R2 names present in backend secrets (r2_access_key, r2_bucket, r2_enabled, r2_endpoint, r2_secret_key); total 25.
- Backend redeployed rev 00024; /api/health 200. _r2_enabled() should be True (all five non-empty).

## NOT yet verified (the boundary)
- R2 *working* is unproven: _r2_enabled() only checks non-empty, NOT that the endpoint is well-formed or creds authenticate (prior 'malformed endpoint' flag means non-empty != working).
- Confirmation requires owner-authed GET /api/upload/_diag OR an upload — both ROB-ONLY. No unauthenticated R2-status route exists.

## Next step — ROB-ONLY (upload boundary)
1. Rob does ONE small test upload (any persona). Then either:
   - GET {backend_base}/api/upload/_diag authed as Rob -> report storage_uri_val + r2_error for that event (scheme + error field, NOT secret values), OR
   - give executor the source_id; executor reads storage_uri scheme read-only from DB.
   backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
2. PASS = storage_uri starts r2:// AND r2_error null. FAIL = file:// or r2_error set -> endpoint/creds still wrong, diagnose; do NOT proceed.
3. On PASS (ROB-ONLY): re-upload ellis videos (or reuse the test upload's persona) -> reprocess.
4. (AUTO-RUN verify) sources advance audio_extracted -> transcribed; source_speaker_segments > 0; btg-stt logs show /stream requests.

## After STT validates
- TTS bridge bring-up (voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval.

## ROB-ONLY (carried)
- Upload/re-upload; authed _diag call; reprocess trigger; TTS/face GPU. No secret values set by executor.

## Hard constraints
No upload/authed-call by executor. No prod DB writes by executor. No reprocess until R2 verified working. No blind retry of file:// ellis sources. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
