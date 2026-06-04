# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Status
STT VALIDATED (HTTP transcribe; 31,583 chars). Two downstream blockers. GATE A drafted (Qwen) — awaiting Rob "apply it". GATE B separate (diarization/pyannote/HF token) — not bundled.

================================================================
## GATE A — persona extraction: Anthropic -> Qwen (DRAFTED, not applied)
================================================================
### Diff (RESULTS 04:36Z) — 1 file backend/app/services/persona_builder.py
1. Imports: remove Anthropic SDK import + module-level Anthropic client; add `from app.services.llm_client import get_llm_client` + `import asyncio`.
2. extract_persona_data: replace client.messages.create(claude-sonnet-4) with get_llm_client("qwen") + await loop.run_in_executor(None, lambda: llm.complete(model="gpt-4o-mini", max_tokens=4096, messages=[{system:EXTRACTION_PROMPT},{user}])). Forces Qwen; QwenLLM maps model id to served model; sync complete in executor (mirrors conversation.py).
3. Read: response_text = (response.choices[0].message.content or "").strip().
- UNCHANGED: EXTRACTION_PROMPT, JSON parsing, []-on-parse-fail, re-raise-on-API-fail.
- No Anthropic anywhere; no ANTHROPIC credential; transcript_text preserved (committed at processing.py:228 before extraction); Gate B untouched.

### Pre-deploy FLAG (must check before redeploy, name-only)
- Backend needs QWEN_BASE_URL + QWEN_API_TOKEN present or QwenLLM raises/conn-fails (would trade Anthropic 401 for Qwen error). Qwen routing verified earlier (D-2) + btg-llm-qwen-4b deployed, but backend's qwen base_url/token presence NOT re-verified for this path.

### Next — ROB DECISION
- "apply it" -> executor applies 3 edits, shows greps + py_compile, commits backtogether. THEN name-only check QWEN_BASE_URL/QWEN_API_TOKEN present (AUTO-RUN). THEN confirm-before-live backend redeploy. THEN Rob reprocess -> verify source completes + PersonaMemory rows created.

================================================================
## GATE B — diarization 0 speaker segments (SEPARATE, not in Gate A)
================================================================
- source_speaker_segments=0 / speaker_embeddings=0. NOT caused by HTTP change (diarize_and_transcribe always returned speakers:{} by design). Real speaker data = pyannote embed_source (processing.py:248).
- STRONG candidate: pyannote needs a HuggingFace token; _inspect showed env_hf_token_present=false. If embed_source needs HF_TOKEN and it's unset, embeddings=0 is explained.
- Next (AUTO-RUN read-only, when Rob wants): confirm whether embed_source requires HF token + whether it logged a [non-fatal] error for 294c44ea (dashboard App Logs). Then ROB decision: set HF token (secret, value never in chat) OR make embeddings optional for STT validation.
- Do NOT bundle into Gate A.

## ROB-ONLY (carried)
- Approve Gate A apply (show-diff done); QWEN secrets presence is name-only check; backend redeploy (confirm-before-live); reprocess; HF token decision (B); push; TTS/face. No secret values read/set by executor. No Anthropic credential (Rob: not using Claude).

## Hard constraints
No code applied without "apply it". No deploy without confirm-before-live. No reprocess/authed-call by executor. No secret values. No push unless asked. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).