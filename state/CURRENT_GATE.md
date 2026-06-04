# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate upload-time STT end-to-end via the HTTP transcribe route. btg-stt deployed + SMOKE PASSED (POST route reachable, 401 not 404). Next: redeploy backend (confirm-before-live), then reprocess a fresh video to validate.

## Done / verified (RESULTS 04:11Z)
- btg-stt redeployed build-8566f671 (HTTP transcribe route live). /healthz 200.
- POST /api/stt/transcribe (no auth) -> 401 "bridge auth failed" NOT 404. PROOF: HTTP POST reaches the container + route works + auth enforced. Option B validated at gateway level. (WS still 404s — irrelevant for upload path.)
- Code committed backtogether 0668e69 (3 files). Realtime WS path intact.

## Next step — ROB-ONLY confirm-before-live
1. Redeploy backend (transcribe_file now httpx POSTs /api/stt/transcribe with X-Bridge-Auth).
   - cmd: cd backend && cerebrium deploy --config-file ./cerebrium.toml (logged bg)
   - smoke: /api/health 200.
2. After backend live (AUTO-RUN verify): Rob reprocess a fresh video [ROB-ONLY trigger] -> executor verifies read-only:
   - source advances audio_extracted -> transcribed.
   - transcript_text populated; source_speaker_segments>0.
   - dashboard btg-stt Runs: POST /api/stt/transcribe 200 (authed call from backend).
   - This is the full end-to-end STT validation (first time STT will have run on a real upload).
   - If transcribe 200 but downstream off (diarization/segments), diagnose from there — no blind retry.

## ROB-ONLY (carried)
- Approve backend redeploy (confirm-before-live); reprocess trigger; push (if wanted); Cerebrium escalation (parallel, for realtime WS); TTS/face. No secret values read/set by executor.

## Hard constraints
No backend deploy without explicit confirm-before-live. No reprocess until backend deployed. No push unless asked. No TTS/face.

## Follow-on
- Realtime-call WS still blocked (gateway WS issue) — separate; upload path does not depend on it.
- TTS/avatar WS bring-up will hit same wall — apply HTTP-vs-WS eval there.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).