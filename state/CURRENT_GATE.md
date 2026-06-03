# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate the STT path end-to-end by re-processing persona ellis, now that the backend is wired to the live btg-stt bridge. STT-only. No TTS, no face/avatar.

## Done so far (verified)
- STT bridge btg-stt: READY, /healthz 200, device=cuda, auth_secret_set=True. (RESULTS 22:50Z)
- Backend wired: stt service url setting present in backend secret store; WS routing to btg-stt /stream confirmed (HTTP 101). (RESULTS 23:04Z)
- Backend redeployed rev 00023 (build-c706e76b); /api/health 200. (RESULTS 23:04Z)
- Before-state: ellis 5 video sources all at audio_extracted with errno-111; source_speaker_segments=0.

## Next step — ROB-ONLY trigger, then AUTO-RUN verify
1. (ROB-ONLY) Rob triggers reprocess for each video source, authed as himself. Reason it is ROB-ONLY: the endpoint mutates prod DB (resets the source row) and requires owner auth (request made as Rob). Executor will NOT call it.
   - endpoint: POST {backend_base}/api/upload/_reprocess/{source_id}
   - backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
   - video source_ids:
     - 1154ccfa-1bb4-4a2a-8e27-62bc3665eb95
     - 368930f7-fdf0-411b-93ee-76a6141c78b9
     - 7fa2e508-aea4-4ecf-ad1d-abe19eddd3e6
     - d9caf45c-81bb-4630-a92d-febf527938a9
     - f149bd58-a66f-462f-a51c-10fe50614783
2. (AUTO-RUN) After Rob triggers, executor verifies read-only:
   - persona_sources video rows advance audio_extracted -> transcribed (or processing_status complete).
   - source_speaker_segments for ellis becomes greater than zero.
   - btg-stt bridge logs show transcription WS requests landing.
   - If a row errors again, capture the exact error (the STT path then needs diagnosis, NOT blind retry).
3. (AUTO-RUN) Write RESULTS + update this gate with outcome.

## After this gate (per standing priority rule; do NOT start without reaching the boundary)
- If reprocess validates: next product-critical step is TTS bridge bring-up (yields voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval at that point.

## ROB-ONLY (carried)
- Reprocess trigger (prod DB mutation + auth as Rob).
- TTS/face GPU deploy = new paid infra.
- No secret values set by executor.

## Hard constraints
No prod DB writes by executor. No calls placed/ended by executor. No backend redeploy without confirm-before-live. No TTS/face this gate.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
