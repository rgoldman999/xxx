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
