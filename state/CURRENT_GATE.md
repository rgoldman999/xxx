# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate STT end-to-end. Blocked by a storage-layer prerequisite: R2 object storage is NOT configured in the backend, so uploads fall back to file:// local disk and do not survive redeploys. R2 must be enabled before any upload-based STT validation.

## Done / verified
- STT bridge btg-stt READY (/healthz 200, cuda, auth_secret_set=True).
- Backend wired + rev 00023; /api/health 200; WS routing to btg-stt /stream confirmed (HTTP 101).
- Reprocess of 5 ellis videos FAILED upstream of STT (FileNotFoundError); sources are file:// local, gone from fresh container.
- Pre-flight for option A found the deeper cause: R2 not enabled (see below).

## ROOT BLOCKER — R2 not configured (RESULTS 23:15Z)
- object_store.py: writes go to R2 only if R2_ENABLED, else file:// fallback.
- _r2_enabled requires r2_enabled + r2_endpoint + r2_access_key + r2_secret_key + r2_bucket.
- config.py defaults: r2_enabled=False, R2 fields empty. Backend secrets contain NO R2 vars.
- => uploads currently land on ephemeral local disk (file://), lost on redeploy. This is why the ellis sources are unretrievable. Re-uploading without R2 repeats the failure.

## Revised path (R2 prerequisite first)
1. (ROB-ONLY) Rob sets backend R2 secrets + enables R2: env names per config.py — r2_endpoint, r2_access_key, r2_secret_key, r2_bucket, r2_enabled=true. Secret VALUES = Rob (never via chat). Per memory, R2 bucket + creds exist in Rob's creds files.
2. (CHECKPOINT / confirm-before-live) backend redeploy to pick up R2 config.
3. (AUTO-RUN verify) read-only: a fresh small upload yields storage_uri starting r2:// (not file://). Confirms R2 active.
4. (ROB-ONLY) re-upload ellis videos (or a test persona video) -> reprocess.
5. (AUTO-RUN verify) sources advance audio_extracted -> transcribed; source_speaker_segments > 0; btg-stt logs show /stream requests.

## After STT validates
- TTS bridge bring-up (voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval.

## ROB-ONLY (carried)
- R2 secret values + enabling R2 (Rob).
- Source upload/re-upload (auth, prod data, R2 writes).
- Reprocess trigger (prod DB mutation + auth).
- Backend redeploy (confirm-before-live).
- TTS/face GPU deploy = new paid infra.
- No secret values set by executor.

## Hard constraints
No secret values set by executor. No prod DB writes / uploads by executor. No backend redeploy without confirm-before-live. No calls placed/ended by executor. No blind retry. No TTS/face this gate.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
