# RESULTS

Append-only log of executor findings/results, written by `handoff write-result`.
Newest entries appended at the bottom. Handoff-doc only; no secrets (the
coordinator refuses to write content matching secret patterns).

## 2026-06-03T22:50:02Z — STT bridge deploy + smoke (btg-stt)
STT bridge deployed and smoke-verified. STT-only; no backend wiring, no TTS/face.

- App: p-a907d7c5-btg-stt — status READY (registered 2026-06-03 22:44:18Z)
- Build: build-c308b402 (clean logged re-deploy; prior attempt was an orphaned hung process from a timed-out call, killed PIDs 27243/27249)
- Endpoint base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt
- /healthz smoke: HTTP 200 (0.3s after one cold-start retry); body ready=true, stt.ready=true, model=deepdml/faster-whisper-large-v3-turbo-ct2, device=cuda, compute_type=int8_float16
- auth_secret_set=True (confirmed in startup log)
- GPU/scale: ADA_L4, scale-to-zero (min0/max1) confirmed — first /healthz timed out on cold start, retry 200; no warm GPU burn while idle
- cuDNN risk did NOT materialize: ctranslate2 loaded on CUDA cleanly (whisper 9.4s, warm pass OK)
- Spend: within $100 ceiling (~$23.7x + minor build/warmup); scale-to-zero, no ongoing GPU cost idle
- Artifacts committed: backtogether 46d523a (local), pyjwt dep added during pre-flight
- STOPPED after smoke per gate. NOT done: backend STT_SERVICE_URL wiring, TTS, face/avatar (each a later separate gate).

## 2026-06-03T23:04:22Z — Backend wired to STT bridge + redeployed (rev 00023); reprocess pending
Backend STT wiring complete and verified. STT path ready to validate via reprocess (Rob action). No TTS, no face/avatar.

DONE + VERIFIED:
- Backend stt service url setting confirmed present in backend secret store (name-only; Rob set it project-scoped). Bridge auth shared value also present.
- WS routing pre-verified: handshake to btg-stt /stream returned HTTP 101 Switching Protocols (Cerebrium/envoy routes WS to bridge 8003 /stream). Backend-constructed wss URL will reach the bridge.
- Backend redeployed: app backtogether-backend revision 00023, build build-c706e76b. Clean startup (uvicorn :8000, worker started). Logged background deploy (no orphan).
- Backend health: GET /api/health -> HTTP 200, {"status":"ok","service":"backtogether"}.

VERIFIED BEFORE-STATE (ellis persona c40776dd, read-only DB):
- 13 persona_sources: 5 video, 8 photo.
- All 5 video rows: processing_stage=audio_extracted, error_message=[Errno 111] Connection refused (the pre-wiring STT failure).
- photo rows: stage=None (no STT).
- source_speaker_segments (ellis): 0.
- video source_ids: 1154ccfa-1bb4-4a2a-8e27-62bc3665eb95, 368930f7-fdf0-411b-93ee-76a6141c78b9, 7fa2e508-aea4-4ecf-ad1d-abe19eddd3e6, d9caf45c-81bb-4630-a92d-febf527938a9, f149bd58-a66f-462f-a51c-10fe50614783.

CHECKPOINT (why stopped): reprocess endpoint POST /api/upload/_reprocess/{source_id} (a) mutates prod DB (resets source row), and (b) requires get_current_user owner auth. Both are ROB-ONLY per rule 5 (prod-data mutation; authenticated request as Rob). Executor will NOT call it. Rob triggers; executor then verifies read-only.

NEXT (Rob action): POST /api/upload/_reprocess/{source_id} for each of the 5 video ids, authed as Rob, against backend base https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend. Then executor verifies (AUTO-RUN, read-only): video rows advance audio_extracted -> transcribed; source_speaker_segments > 0; STT bridge logs show transcription requests.

NOT done: reprocess (Rob), TTS bridge, face/avatar. Spend within ceiling.
