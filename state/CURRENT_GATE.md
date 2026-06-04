# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Status
STT transcription VALIDATED end-to-end (HTTP /api/stt/transcribe; transcript 31,583 chars). Upload pipeline now has TWO separate downstream blockers, both characterized read-only (RESULTS 04:33Z). Neither is caused by the STT/HTTP work.

================================================================
## GATE A — persona extraction wrongly hardcoded to Anthropic (FATAL)
================================================================
### Finding
- persona_builder.py hardcodes the Anthropic SDK (module-level client, model "claude-sonnet-4-20250514") and re-raises on API failure -> the 401 invalid-key fails the whole source at processing.py:301.
- Rob: the app is NOT supposed to use Claude/Anthropic. So the fix is NOT to set an Anthropic key — it is to route extraction through the app's provider-agnostic client.
- The app already has get_llm_client() (llm_client.py): .complete(**kwargs), OpenAI SDK shape, Qwen via LLM_PROVIDER=qwen (default OpenAI). conversation.py already uses it.
### Proposed fix (DRAFT only; AUTO-RUN draftable + show-diff; deploy confirm-before-live)
- Rewrite extract_persona_data to use get_llm_client().complete(...) (mirror conversation.py: run_in_executor + resp.choices[0].message.content). Remove anthropic import + hardcoded model + module-level client. Keep re-raise-on-API-fail / []-on-parse-fail semantics.
### ROB DECISION needed
- Which provider should persona extraction use: Qwen (LLM_PROVIDER=qwen, per stack decision #19) or OpenAI (llm_client default)? Code becomes provider-agnostic either way; this only sets the model id / provider env.
- Then: "draft it" -> executor produces show-diff -> apply on approval -> confirm-before-live backend redeploy -> re-trigger reprocess.

================================================================
## GATE B — diarization produced 0 speaker segments (SEPARATE, pre-existing)
================================================================
### Finding
- source_speaker_segments=0, speaker_embeddings=0, speech_speakers=0.
- NOT caused by the HTTP change: diarize_and_transcribe ALWAYS returned speakers:{} by design (docstring); speaker data comes from a separate pyannote path: voice_clusterer / embed_source (processing.py:248).
- So embed_source either genuinely found 0 speakers in this file OR raised non-fatally (processing.py:283-288 catches "[non-fatal] embed_source raised"). Job reached stage=clustered, so embedding ran and yielded 0.
### Next (AUTO-RUN read-only)
- Inspect whether embed_source logged a [non-fatal] error for 294c44ea (needs backend App Logs — flaky CLI, use dashboard) and whether pyannote/HF token (env_hf_token_present was false in _inspect) is required for embeddings. NOTE _inspect earlier showed env_hf_token_present=false -> pyannote diarization/embedding may need a HF token that is not set. STRONG candidate for why embeddings=0.
- Determine if 0 speakers is expected (e.g. this file's audio) or a config gap (missing HF token for pyannote).
### Likely sub-finding to verify
- pyannote.audio speaker embeddings typically require a HuggingFace token (accept model terms). _inspect reported env_hf_token_present=false. If embed_source needs HF_TOKEN and it is unset, embeddings=0 is explained -> ROB sets HF token (secret, value never in chat) OR confirm embeddings are optional for STT validation.

## Recommended order
- GATE A first (fatal, blocks the job). GATE B can be investigated read-only in parallel; its fix (likely HF token) is independent.

## ROB-ONLY (carried)
- Provider decision (A); any secret value incl HF token (B); approve code change (show-diff); backend redeploy (confirm-before-live); reprocess; push; TTS/face. No secret values read/set by executor.

## Hard constraints
No code change without show-diff + approval. No deploy without confirm-before-live. No reprocess/authed-call by executor. No secret values. No push unless asked. No TTS/face. Do NOT ask Rob for an Anthropic credential (Rob: not using Claude).

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).