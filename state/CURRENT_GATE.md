# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
STT transcription is VALIDATED end-to-end (HTTP /api/stt/transcribe). The reprocess job then failed on two downstream issues: (1) Anthropic 401 invalid x-api-key, (2) source_speaker_segments=0. Resolve those next. Upload pipeline not yet fully green.

## Done / verified (RESULTS 04:28Z)
- STT PASS: transcript_text = 31,583 chars of real speech via the new HTTP path. Source cleared audio_extracted -> transcribed -> clustered. Job 8370a8d7 ran on rev 00026. The WS->HTTP fix WORKS. (First successful transcription on a real upload.)
- Job 8370a8d7 FINAL: failed, stage=clustered, retry 1, completed 04:27:39.

## Two downstream blockers (NEW, separate from STT)
1. ANTHROPIC 401 (ROB-ONLY secret): error = AuthenticationError 401 'invalid x-api-key' (req_011C... = Anthropic) at processing.py:301 extract_persona_data. Backend ANTHROPIC_API_KEY secret is invalid/expired. Rob must set a valid key (value never in chat), redeploy backend, re-verify.
   - NOTE: the locked stack uses Qwen for LLM, but this upload-extraction path calls Anthropic directly. Worth Rob confirming whether extract_persona_data SHOULD use Anthropic or should route to Qwen (possible stack-decision mismatch) — flag, not fix.
2. source_speaker_segments=0 / speaker_embeddings=0 (diarization path): transcript is full but the pyannote/diarization side produced nothing. Verification condition segments>0 NOT met. Open question: does the new HTTP /api/stt/transcribe return only text (not the speakers{} dict the old WS diarize_and_transcribe provided), or is diarization a separate pyannote step (processing.py:240-248 embed_source) that zeroed independently? Needs read-only review of diarize_and_transcribe return shape vs the new transcribe_file. (May mean the HTTP route needs to also return/representation diarization, OR embed_source failed before the Anthropic error.)

## Next step — Rob decision / ordering
A. (ROB-ONLY) Set a valid ANTHROPIC_API_KEY (or confirm extract_persona_data should use Qwen, not Anthropic). Then backend redeploy (confirm-before-live) + re-trigger reprocess.
B. (AUTO-RUN read-only, can do now) Investigate the segments=0 / diarization question: compare diarize_and_transcribe (does it still return speakers{}?) vs the new transcribe_file return shape; determine if HTTP path dropped diarization or pyannote embed_source failed independently. Report; propose fix as draft (gated).
- Recommend B now (read-only, no waiting) to characterize blocker 2 while Rob handles the key for blocker 1.

## ROB-ONLY (carried)
- ANTHROPIC_API_KEY value; Qwen-vs-Anthropic routing decision; backend redeploy; reprocess; push; TTS/face. No secret values read/set by executor.

## Hard constraints
No reprocess/deploy/authed-call by executor. No prod DB writes. No secret values. No push unless asked. No TTS/face.

## Follow-on
- Realtime WS still blocked (separate). TTS/avatar WS -> same HTTP treatment at bring-up.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).