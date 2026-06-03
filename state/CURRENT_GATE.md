# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Option 1 — wire the backend to the live btg-stt bridge, redeploy backend, re-process persona ellis to validate the STT path end-to-end. (STT bridge itself = DONE/verified.) No TTS, no face/avatar.

## Verified inputs (read-only, 2026-06-03)
- STT endpoint base (live, healthz returns 200): the btg-stt app https base at api.aws.us-east-1.cerebrium.ai under v4 / p-a907d7c5 / btg-stt.
- Backend reads its stt service url setting (config.py line 61); client stt_bridge.py _ws_url_from converts an https base into a wss host /stream URL with the bridge auth query appended.
- WS ROUTING CONFIRMED: a handshake to the btg-stt /stream path returned HTTP 101 Switching Protocols (Cerebrium/envoy routes WebSocket to the bridge on 8003 /stream). The constructed wss URL will reach the bridge.
- Backend secret store: the bridge auth shared value is present (project-scoped, shared with the bridge). The stt service url setting is NOT yet present.

## Required setting
- The backend stt service url setting must be set to the btg-stt https base (the client appends /stream and upgrades to wss). ROB-ONLY: Rob sets this value in the secret store. It is a plain URL, not a credential, but per rule 5 it is a secret-store value Rob provides and it changes prod behavior.
- Bridge auth shared value: already present on backend + bridge. No action.

## Plan
1. (ROB-ONLY) Rob sets the backend stt service url setting to the btg-stt https base, via CLI or dashboard. Confirm present name-only.
2. (CHECKPOINT / confirm-before-live) Redeploy backend to pick up the new setting. Backend is the user-facing prod service, so its redeploy is confirm-before-live, not auto.
   - target: project p-a907d7c5, app backtogether-backend, us-east-1
   - command: from repo root, cerebrium deploy against backend/cerebrium.toml (confirm exact path at deploy)
   - smoke: backend health returns 200; then re-process persona ellis and confirm the STT stage advances past the prior errno-111 connection-refused.
   - rollback: previous backend revision remains available; redeploy prior build on regression. Backend has live users, so stop on any health regression.
3. (AUTO-RUN) Re-process persona ellis (c40776dd): trigger source reprocess; verify persona_sources video rows advance from audio_extracted to transcribed and that source_speaker_segments becomes greater than zero. Read-only DB verification of stage progression.
4. (AUTO-RUN) Update gate + RESULTS.

## ROB-ONLY stops (per standing rule 5)
- Setting the backend stt service url value (secret-store value Rob provides).
- Backend redeploy changes prod behavior for all users -> confirm-before-live.
Everything else (verify endpoint, plan, reprocess trigger if non-secret, DB read, handoff docs) = AUTO-RUN.

## NOT this gate
TTS bridge, face/avatar, any new GPU infra.

## Hard constraints
No backend deploy without Rob confirm-before-live. No secret values set by executor. No prod DB writes beyond an approved reprocess. No calls placed/ended by executor.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
