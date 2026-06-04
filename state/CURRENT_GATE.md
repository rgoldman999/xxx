# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Unblock upload-time STT via an HTTP POST transcribe endpoint (sidesteps the WS gateway 404, which is confirmed app-wide on btg-stt). Draft DONE + verified against source. Awaiting Rob approval to apply + deploy.

## Why HTTP (recap)
Upload reprocess = batch transcription of a finite .wav -> no streaming needed. POST /api/stt/transcribe routes as normal HTTP (gateway proxies it; /healthz GET 200 proves HTTP works), avoiding the WS-upgrade-not-proxied bug entirely. Realtime-call WS is a separate, still-open concern (not this gate).

## Drafted (RESULTS 02:37Z) — 3 edit sites, verified, NOT applied
1. bridge/services/stt.py + ops/cerebrium/stt/services/stt.py (snapshot):
   - imports: add Request, Depends + `from auth import require_bridge_auth` (HTTP header auth; stt.py already imports require_bridge_auth_ws at line 64 — path confirmed).
   - add _transcribe_file_blocking(path, language) (faster-whisper decodes path, vad_filter=True).
   - add @router.post("/transcribe", Depends(require_bridge_auth)) reading raw request.body() (no multipart -> no missing python-multipart dep), temp file, transcribe on thread, return {text,duration_s}. Lands at /api/stt/transcribe (prefix already committed d30301e).
2. backend stt_bridge.transcribe_file (line 248): rewrite to httpx POST /api/stt/transcribe, params={language}, content=<bytes>, X-Bridge-Auth header (house pattern), timeout=1800. Returns text; raises on failure.
   - KEEP WS /stream + open_stream (realtime path); change ONLY transcribe_file.

## Verified assumptions (read-only)
- HTTP auth = header X-Bridge-Auth (auth.py:44), not ?query (corrected). Client sends header.
- Backend uses httpx (not aiohttp). Mirrors tts/face bridge.
- Raw body avoids missing python-multipart in bridge toml.
- faster-whisper transcribes a file path directly.
- POST route reachability proven by /healthz GET 200 + working backend HTTP.

## Next step — ROB DECISION
"apply it" -> executor: apply 3 edits, show greps/diff, commit (backtogether), then confirm-before-live redeploy btg-stt FIRST (new route exists) THEN backend (client posts to it). Then Rob reprocess a fresh video -> executor verifies read-only: audio_extracted -> transcribed, source_speaker_segments>0, dashboard /api/stt/transcribe POST 200.

## ROB-ONLY (carried)
- Approve apply + commit; redeploy (confirm-before-live, btg-stt then backend); upload/reprocess; Cerebrium escalation (parallel, optional); TTS/face. No secret values read/set by executor.

## Hard constraints
No edits applied / no commit without "apply it". No deploy without confirm-before-live. No reprocess until both deployed. No push unless asked. No TTS/face.

## Follow-on
- Realtime-call WS (/stream) still blocked by the same gateway issue — needs Cerebrium answer or a non-WS realtime design later. Upload path does NOT depend on it.
- TTS/avatar bridges also stream over WS -> will hit the same gateway wall; same HTTP-vs-WS evaluation applies at their bring-up.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).