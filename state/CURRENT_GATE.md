# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
STT bridge bring-up — stand up the STT bridge as a direct Cerebrium GPU app (faster-whisper large-v3-turbo + silero VAD) and smoke-test it in isolation. STT ONLY. (Coordinator Phase 1 = DONE, commit 256cbd9. TTS / face-avatar = NOT this gate.)

## ROB-ONLY items (explicit — never auto, never gate-file-authorized)
- Rob sets BRIDGE_AUTH_SECRET himself (openssl rand -hex 32 → cerebrium secret set / dashboard). Value NEVER in chat. Project-scoped (p-a907d7c5) covers STT app + backend.
- Rob confirms the STT config (L4 / scale-to-zero) before deploy.
- Confirm-before-live at the moment of deploy (first paid GPU infra).
- Rob accepts the bounded STT GPU spend (≤ $100, scale-to-zero).

## AUTO-RUN allowed this gate
- commit held STT deploy artifacts (ops/cerebrium/stt/cerebrium.toml) to backtogether
- inspect STT config + bridge source files (read-only pre-flight)
- prepare exact secret-setting instructions (no values exposed)
- prepare deploy command / smoke / rollback (recorded below)
- update handoff docs (CURRENT_GATE.md / RESULTS.md) and commit/push

## DO NOT (this gate)
- Do NOT deploy yet (wait: secret confirmed present + Rob's confirm-before-live).
- Do NOT set secret values.
- Do NOT start GPU spend yet.
- Do NOT start TTS or face/avatar.
- Do NOT change backend STT_SERVICE_URL.
- Do NOT deploy backend.
- Do NOT retry persona processing.
- No new paid infra beyond the single STT app.

## STT deploy spec (for the gated deploy, once secret confirmed + Rob OK)
- target: Cerebrium project p-a907d7c5, app name `btg-stt`, region us-east-1
- scope: single GPU app from ops/cerebrium/stt/cerebrium.toml — STT bridge only; no backend change, no other app
- command: `cd ~/Projects/backtogether/ops/cerebrium/stt && cerebrium deploy --config-file ./cerebrium.toml`
- config: compute ADA_L4, gpu_count 1, min_replicas 0 (scale-to-zero), max_replicas 1, port 8003, healthcheck /healthz, base nvidia/cuda:12.4.1-cudnn-runtime, apt ffmpeg
- smoke: after "live", GET <app https base>/healthz → expect 200 with ready=true (whisper+vad loaded) and (from startup log) auth_secret_set=true
- rollback / stop condition: if build fails OR /healthz not ready OR auth_secret_set=false OR spend behavior unclear → STOP, do not wire backend, do not retry blindly; capture exact error; scale-to-zero means idle cost ≈ 0 while diagnosing. App can be deleted (`cerebrium apps delete btg-stt`) to halt entirely — that deletion is reversible re-deploy, no prod data involved.
- precondition: BRIDGE_AUTH_SECRET present (name-only confirm via `cerebrium secrets list`) BEFORE deploy.

## Next-step sequence
1. (Rob) set BRIDGE_AUTH_SECRET; confirm present (name-only).
2. (AUTO-RUN) read-only pre-flight: confirm ops/cerebrium/stt include paths exist + bridge imports resolve.
3. (AUTO-RUN) commit ops/cerebrium/stt artifacts to backtogether (show diff in report).
4. (CHECKPOINT) confirm-before-live → deploy btg-stt → smoke /healthz → report endpoint+status.
5. (AUTO-RUN) update CURRENT_GATE.md + RESULTS.md; commit handoff docs.
STOP after STT smoke. Backend wiring (STT_SERVICE_URL) + TTS are later, separate gates.

## Hard constraints (unchanged)
No backend deploy. No env/secret values set by executor. No prod DB writes. No calls placed/ended by executor. App-repo commits show-diff in report.

---

## D-2 ARCHIVE — CLOSED (verified w/ formatter caveat)
call_id e1f9cdd4-6abc-4469-81b7-4d69637fc6fa; first prod calls row for c447365d…; llm_provider_route_decision EMITTED (call.py:582); basicConfig drops extra={} so literal override=qwen not greppable; verified via chain (row+resolver+0 non-allowlisted). Active call e1f9cdd4… status=active to be ended (user-run).

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear broad-scope wrangler OAuth session on Mac.
