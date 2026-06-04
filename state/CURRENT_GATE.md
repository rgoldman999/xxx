# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Full end-to-end upload STT validation. Both apps deployed on the HTTP transcribe path. Awaiting Rob's reprocess trigger; then executor verifies read-only.

## Done / verified (RESULTS 04:16Z)
- btg-stt build-8566f671: HTTP /api/stt/transcribe live, POST smoke 401 (reachable, auth enforced).
- backend rev 00026: /api/health 200; transcribe_file httpx POSTs /api/stt/transcribe + X-Bridge-Auth.
- Code committed backtogether 0668e69 (not pushed). Realtime WS path intact.

## Next step — ROB-ONLY trigger, then AUTO-RUN verify
1. Rob triggers reprocess of a fresh video. Recommended: 294c44ea-9784-42eb-988a-701a11d7c448 (already r2:// + audio_extracted -> goes straight to transcribe step). POST {backend_base}/api/upload/_reprocess/294c44ea... authed. backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend. (35-min file; transcription takes time.)
2. (AUTO-RUN read-only verify):
   - audio_extracted -> transcribed; transcript_text populated; source_speaker_segments>0.
   - btg-stt dashboard Runs: authed POST /api/stt/transcribe 200.
   - = first successful end-to-end STT on a real upload.
3. If transcribe 200 but diarization/segments off -> diagnose (no blind retry).

## After STT validates end-to-end
- This unblocks the upload pipeline. Next product-critical = TTS bridge bring-up (voice_id / callable persona). New paid GPU = ROB-ONLY spend. NOTE: TTS streams over WS -> will hit the gateway WS wall; plan HTTP transport for TTS too (mirror this fix) before/at bring-up.

## ROB-ONLY (carried)
- Reprocess trigger; push (if wanted); Cerebrium escalation (realtime WS, parallel); TTS/face GPU. No secret values read/set by executor.

## Hard constraints
No reprocess/authed-call by executor. No prod DB writes by executor. No push unless asked. No TTS/face.

## Follow-on
- Realtime-call WS (/stream) still blocked by gateway WS issue — separate; needs Cerebrium answer or non-WS realtime design.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).