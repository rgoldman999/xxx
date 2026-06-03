# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
STT bridge bring-up = DONE (deployed + smoke-verified). Awaiting Rob's choice of next gate.

## STT result — COMPLETE (verified)
- App p-a907d7c5-btg-stt READY; build build-c308b402.
- Endpoint base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt
- /healthz: HTTP 200, ready=true, stt.ready=true, model=deepdml/faster-whisper-large-v3-turbo-ct2, device=cuda, compute_type=int8_float16; auth_secret_set=True (startup log).
- ADA_L4, scale-to-zero (min0/max1) confirmed (cold-start on first hit, then 200). Spend within $100.
- See RESULTS.md 2026-06-03T22:50Z for full detail.

## NOT done (each a later, separately-approved gate)
- Backend STT_SERVICE_URL wiring (would change backend behavior → its own gate).
- TTS bridge bring-up (needed for voice_id / callable persona).
- Face/avatar bridge (avatar mode only).

## Next gate — Rob chooses ONE
1. Wire backend STT_SERVICE_URL to the btg-stt endpoint + re-process persona (CHECKPOINT/ROB-ONLY: changes backend, new secret value STT_SERVICE_URL, backend redeploy).
2. TTS bridge bring-up (mirror the STT path: artifacts → secret already set project-scoped → deploy → smoke). New paid GPU infra → ROB-ONLY spend approval.
3. Hold / stop session.

## ROB-ONLY (carried)
- Any backend STT_SERVICE_URL set = secret value + changes prod behavior → ROB-ONLY.
- TTS/face GPU deploy = new paid infra → ROB-ONLY spend approval.
- BRIDGE_AUTH_SECRET already set project-scoped (covers future bridges).

## Hard constraints (unchanged)
No backend deploy/wiring without explicit approval. No env/secret values set by executor. No prod DB writes. No calls placed/ended by executor.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear broad-scope wrangler OAuth session on Mac. Active test call e1f9cdd4... still status=active (end user-run).
