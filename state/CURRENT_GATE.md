# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Deploy the HTTP transcribe fix and validate upload-time STT end-to-end. Code APPLIED + COMMITTED (backtogether 0668e69), not deployed. Awaiting confirm-before-live — btg-stt FIRST, then backend.

## Done (RESULTS 02:41Z)
- Added POST /api/stt/transcribe to bridge (stt.py + snapshot, byte-identical); backend transcribe_file now httpx POSTs it with X-Bridge-Auth. py_compile OK all 3; only 3 files changed; realtime WS /stream + open_stream intact.
- Commit backtogether main 0668e69, clean, NOT pushed.

## Next step — ROB-ONLY confirm-before-live, IN ORDER
1. Redeploy btg-stt FIRST (route must exist first).
   - cmd: cd ops/cerebrium/stt && cerebrium deploy --config-file ./cerebrium.toml (logged bg, ~3-4 min)
   - smoke: /healthz 200; then POST /api/stt/transcribe (tiny wav + X-Bridge-Auth header) -> 200 JSON {text,duration_s}; dashboard Runs shows POST /api/stt/transcribe 200 NOT 404.
   - HALT if POST 404s -> deeper gateway issue (unexpected since HTTP works); do not deploy backend.
2. THEN redeploy backend (client posts to new route). smoke /api/health 200.
3. After both live (AUTO-RUN verify): Rob reprocess a fresh video [ROB-ONLY] -> source advances audio_extracted -> transcribed, source_speaker_segments>0, transcript_text populated, dashboard /api/stt/transcribe POST 200.

## ROB-ONLY (carried)
- Approve each redeploy (confirm-before-live, separately); upload/reprocess; push (if wanted); Cerebrium escalation (parallel/optional, for realtime WS); TTS/face. No secret values read/set by executor.

## Hard constraints
No deploy without explicit confirm-before-live (btg-stt and backend separately). No reprocess until both deployed + POST smoke passes. No push unless asked. No TTS/face.

## Follow-on
- Realtime-call WS (/stream) still blocked by the gateway WS issue — separate; needs Cerebrium answer or non-WS realtime design. Upload path (this fix) does not depend on it.
- TTS/avatar bridges stream over WS -> same gateway wall expected; apply same HTTP-vs-WS evaluation at their bring-up.
- /api/stt prefix commit d30301e: now load-bearing (route lives at /api/stt/transcribe).

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).