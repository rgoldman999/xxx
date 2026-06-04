# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Status
STT VALIDATED. Gate A (persona extraction -> Qwen) APPLIED + committed (backtogether 890c95d), NOT deployed. Awaiting confirm-before-live backend redeploy. Gate B (diarization) separate.

## Gate A — done / verified (RESULTS 04:41Z)
- persona_builder.py: Anthropic removed; extract_persona_data -> get_llm_client("qwen") + run_in_executor(llm.complete(...)); reads choices[0].message.content. py_compile OK; only this file changed; diff = approved draft. Commit 890c95d (not pushed).
- Pre-deploy check (name-only): QWEN_BASE_URL + QWEN_API_TOKEN present in backend secrets. (Names only; value-valid/reachable confirmed only by the post-deploy reprocess.)

## Next step — ROB-ONLY confirm-before-live
1. Redeploy backend: cd backend && cerebrium deploy --config-file ./cerebrium.toml (logged bg). Smoke: /api/health 200.
2. After live: Rob reprocess 294c44ea (authed) -> executor verifies read-only:
   - source status NOT failed (completes); error_message empty.
   - PersonaMemory rows created for persona 14578822 (Qwen extraction ran).
   - transcript_text still populated.
   - If Qwen call fails (conn/auth/timeout): capture exact error -> Qwen-endpoint config issue, diagnose (no blind retry).

## Gate B — diarization 0 segments (SEPARATE, parked)
- speaker_segments=0 / speaker_embeddings=0. pyannote embed_source path; strong candidate = missing HF token (env_hf_token_present=false in _inspect). Investigate read-only when Rob wants; fix (HF token secret OR make embeddings optional) is independent of Gate A. NOT in this deploy.

## ROB-ONLY (carried)
- Approve backend redeploy (confirm-before-live); reprocess; HF-token decision (B); push; TTS/face. No secret values read/set by executor. No Anthropic credential.

## Hard constraints
No deploy without confirm-before-live. No reprocess/authed-call by executor. No prod DB writes. No secret values. No push unless asked. No TTS/face.

## Cosmetic follow-up (non-blocking)
- persona_builder.py line ~68 has a stale comment ("Claude returned but the shape...") in the parse-failure path; behavior unaffected. Tidy later.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).