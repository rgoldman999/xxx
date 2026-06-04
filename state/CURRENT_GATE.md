# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Exercise the STT bridge end-to-end: get a VIDEO source into R2, reprocess it, and verify transcription. R2 prerequisite is now COMPLETE. No video has yet reached the STT bridge, so STT is still unvalidated end-to-end.

## Done / verified (read-only)
- STT bridge btg-stt READY (/healthz 200, cuda, auth_secret_set=True); WS routing to /stream confirmed (HTTP 101).
- Backend wired (stt service url set), rev 00025, /api/health 200.
- R2 WORKING: fresh upload 066286e3 (01:12Z, rev 00025) -> storage_uri r2://, completed. (Before/after: <=00:38 file://; 01:12 r2://.) RESULTS 01:13Z.

## Next step — ROB-ONLY trigger, then AUTO-RUN verify
1. (ROB-ONLY) Rob uploads a fresh VIDEO source (new persona or re-upload to ellis) — now lands r2://. Do NOT reuse the old file:// ellis videos (not in R2). Then trigger reprocess for that video: POST {backend_base}/api/upload/_reprocess/{source_id}, authed as Rob. backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
2. (AUTO-RUN) executor verifies read-only:
   - the video source advances audio_extracted -> transcribed (or processing_status complete).
   - source_speaker_segments for that persona > 0.
   - btg-stt logs show /stream transcription WS requests landing.
   - capture exact error if it fails (then diagnose, NOT blind retry).
3. (AUTO-RUN) write RESULTS + update gate with outcome.

This is the step that finally exercises the STT bridge end-to-end. PASS here = STT validated.

## After STT validates
- TTS bridge bring-up (yields voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval. Mirrors the STT bring-up path (artifacts -> BRIDGE_AUTH_SECRET already project-scoped -> deploy -> smoke).

## ROB-ONLY (carried)
- Video upload + reprocess trigger (auth as Rob, prod data/DB). TTS/face GPU = new paid infra. No secret values read/set by executor.

## Hard constraints
No upload/authed-call/reprocess by executor. No prod DB writes by executor. No blind retry of file:// ellis sources. No TTS/face this gate.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
