# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Gate 2 — Bridge-tier bring-up. Stand up STT, then TTS, then face/avatar so a persona reaches callable (real voice_id) and a real call works end-to-end. (D-2 route verification already CLOSED — see D-2 ARCHIVE below.)

## Status of inputs (verified read-only 2026-06-03)
- Bridge code EXISTS in repo: bridge/main_stt.py (PORT 8003, faster-whisper+silero, /stream WS), bridge/main_tts.py (PORT 8001 in code header / backend expects 8002 — RECONCILE), bridge/main_av.py (PORT 8000, face+avatar routers). Shared bridge/auth.py (BRIDGE_AUTH_SECRET), sdk_pool.py, services/{stt,tts,face,avatar}.py.
- NO deploy artifacts in bridge/: no Dockerfile, no cerebrium.toml, no requirements.txt. Bring-up must create these.
- Cerebrium apps present: ONLY backtogether-backend + btg-llm-qwen-4b. No STT/TTS/face app.
- Backend env names the bridges wire to (config.py:57-71): stt_service_url, tts_service_url, face_service_url, avatar_service_url, bridge_base_url (legacy), bridge_auth_secret, runpod_api_key. All default to localhost; none set as secrets.
- PRIOR ATTEMPT (STATE 2026-05-05): RunPod SERVERLESS btg-stt image crashed exit-1 at boot, no stderr. Rob: "no serverless." Pivoted to bridge as plain FastAPI on a direct GPU pod (RTX 4090 24GB ~$0.69/hr). => serverless is a known dead-end; use a direct GPU app/pod pattern.

## Bridge-tier bring-up plan (DRAFT — planning only, nothing deployed)

### Services needed (in order)
1. STT — faster-whisper large-v3-turbo + silero VAD. GPU. Serves WS /stream on :8003. Unblocks diarize_and_transcribe (the current [Errno 111]).
2. TTS — Chatterbox voice clone + synth. GPU. Serves :8002 (backend expects 8002; code header says 8001 — pin to 8002). Produces voice_id (voice clone) → makes persona callable.
3. Face/Avatar — AuraFace embed + Ditto avatar. GPU. Serves :8000. Only needed for avatar-mode calls; audio calls need STT+TTS only.

### Deploy order + rationale
STT first (clears current blocker, smallest surface) → TTS (the piece that actually yields voice_id/callable) → Face/Avatar last (avatar mode only). Audio-mode callable persona = STT + TTS. Validate each before the next.

### Exact secrets/env names needed (backend side, set AFTER each bridge is up)
- STT_SERVICE_URL = <stt app https base>           (ROB-ONLY: secret value)
- TTS_SERVICE_URL = <tts app https base>            (ROB-ONLY: secret value)
- FACE_SERVICE_URL / AVATAR_SERVICE_URL = <av base> (ROB-ONLY: secret value; face stage)
- BRIDGE_AUTH_SECRET = <shared secret>              (ROB-ONLY: secret value; set on BOTH backend and each bridge)
- (RUNPOD_API_KEY only if RunPod path — NOT recommended per prior dead-end)
Bridge side each needs: BRIDGE_AUTH_SECRET, PORT, device/CUDA env (STT_DEVICE, LD_LIBRARY_PATH for cuDNN per main_stt.py header).

### Smoke tests (per service, before wiring backend)
- STT: health 200; WS /stream?bridge_auth=… accepts; feed a short wav → returns transcript text.
- TTS: health 200; clone from ~10s ref → returns voice_id; synth a line → audio bytes.
- Face/Avatar: health 200; /face embed on a photo → embedding; avatar first-frame.
- End-to-end: re-process persona ellis (c40776dd) → persona_sources video rows go past audio_extracted to transcribed/clustered; voice_cloner sets voice_id; persona shows callable; UI Call button appears.

### Cost / spend checkpoint (ROB-ONLY)
Each bridge is a GPU service = new paid persistent infra. Prior pattern ~$0.69/hr per RTX-4090-class pod; 3 services (or a shared GPU) = ongoing spend. Exact Cerebrium GPU pricing + whether min_replicas>0 (always-on) vs scale-to-zero must be priced before launch. THIS IS A ROB-ONLY GATE.

## CHECKPOINT / ROB-ONLY boundary for this gate
Planning above is AUTO-RUN (done). The NEXT actions cross ROB-ONLY and STOP here:
- Deploying any bridge GPU app = new paid persistent infra/spend → ROB-ONLY.
- Setting *_SERVICE_URL / BRIDGE_AUTH_SECRET = secret values Rob provides → ROB-ONLY.
=> First ROB-ONLY item: **approve standing up the STT bridge as a GPU app on Cerebrium (or chosen platform), accepting the ongoing GPU spend, and provide/confirm BRIDGE_AUTH_SECRET.** Nothing deploys until Rob approves spend + provides the secret.

## First ROB-ONLY item requiring Rob
Approve STT bridge GPU deployment + ongoing GPU spend, and provide BRIDGE_AUTH_SECRET value. (Plus decide platform: direct Cerebrium GPU app vs other; serverless deprecated per prior failure.)

## Open prerequisites to resolve before/with STT deploy (AUTO-RUN to draft, none done)
- Create bridge deploy artifacts (Dockerfile/cerebrium.toml/requirements) — net-new files; drafting = AUTO-RUN, deploying = ROB-ONLY.
- Reconcile TTS port (code 8001 vs backend-expected 8002).
- GPU image must satisfy main_stt.py cuDNN note (ctranslate2 cuDNN8 vs torch cuDNN9).

## Hard constraints (unchanged)
No deploy. No code changes. No env/secret changes. No bridge work beyond planning. No prod DB writes. No calls placed/ended by executor. Show diff and hold before any commit that isn't handoff-doc-only.

---

## D-2 ARCHIVE — CLOSED (verified with formatter caveat)
Direct API/WS test by USER 2026-06-03. call_id e1f9cdd4-6abc-4469-81b7-4d69637fc6fa; POST /api/call/start 200 → first prod calls row for c447365d… (persona c40776dd, audio, active, 20:58:10Z); WS /api/call/ws/e1f9cdd4… accepted; llm_provider_route_decision EMITTED (call.py:582). basicConfig drops extra={} so literal override=qwen not in log text; verified via chain (row user_id=c4473 + resolver returns qwen for c4473 + 0 non-allowlisted calls). Downstream: WS then errored at call.py:629 on TTS bridge localhost:8002 (bridge absent) — does NOT affect D-2 (route-log fires first). Open: active call e1f9cdd4… status=active should be ended (user-run).
