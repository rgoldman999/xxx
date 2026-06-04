# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Status
STT VALIDATED. Gate A (persona extraction -> Qwen) APPLIED + DEPLOYED (backend rev 00027, /api/health 200). Awaiting Rob reprocess to validate extraction end-to-end. Gate B (diarization) separate.

## Gate A — deployed (RESULTS 04:47Z)
- backend rev 00027 live; carries commit 890c95d (persona_builder -> get_llm_client(qwen), Anthropic removed). /api/health 200. QWEN_BASE_URL/QWEN_API_TOKEN present (name-only).

## Next step — ROB-ONLY trigger, then AUTO-RUN verify
1. Rob reprocess 294c44ea-9784-42eb-988a-701a11d7c448 (authed POST /api/upload/_reprocess/...). Report HTTP status/body.
2. (AUTO-RUN read-only verify):
   - source status completes (NOT failed); error_message empty (no Anthropic 401).
   - PersonaMemory rows created for persona 14578822 (Qwen extraction ran).
   - transcript_text still populated.
   - btg-stt Runs: POST /api/stt/transcribe 200 (authed, from backend).
   - If Qwen call fails: capture exact error (conn/auth/timeout/parse) -> diagnose, no blind retry.
   - NOTE: speaker_segments will likely still be 0 — that is GATE B, expected, not a Gate A failure.

## Gate B — diarization 0 segments (SEPARATE, parked)
- speaker_segments=0 / speaker_embeddings=0; pyannote embed_source; strong candidate = missing HF token (env_hf_token_present=false). Investigate read-only on request; fix independent of Gate A.

## ROB-ONLY (carried)
- Reprocess trigger; HF-token decision (B); push; TTS/face. No secret values read/set by executor.

## Hard constraints
No reprocess/authed-call by executor. No deploy (Gate A deploy done). No prod DB writes. No secret values. No push unless asked. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).