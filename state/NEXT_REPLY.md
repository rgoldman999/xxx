# NEXT REPLY

status: PENDING
updated_at: 2026-06-04T04:45:00Z
consumed_at:
consumed_by:
gate_commit: 5495e05
classification_hint: APPLY approved code patch; deploy remains confirm-before-live

## body
Rob says: apply it.

Approval scope:
- Apply Gate A persona-extraction fix using Qwen/provider-agnostic LLM client.
- persona_builder.py should stop using the raw Anthropic SDK / Claude model directly.
- Use the existing provider-agnostic get_llm_client().complete(...) path, consistent with conversation.py and the existing Qwen/OpenAI provider setup.
- Target provider should be Qwen per the current stack direction; keep the implementation provider-agnostic so env/config can select the provider.
- Do not require ANTHROPIC_API_KEY.
- Preserve transcript_text and prevent downstream Anthropic 401 from killing STT validation.

Execution instructions:
1. Apply the minimal code change that was drafted for Gate A.
2. Show post-edit diff/greps.
3. Run targeted local tests or syntax checks if available.
4. Commit code changes to the product repo.
5. Update RESULTS.md and CURRENT_GATE.md.
6. Stop before deploy and request confirm-before-live backend redeploy.

Keep Gate B separate:
- source_speaker_segments=0 / speaker_embeddings=0 remains a separate diarization/pyannote/HF-token issue.
- Do not bundle Gate B into the persona extraction fix.

Hard constraints:
- No backend deploy until confirm-before-live.
- No reprocess/upload/retry.
- No TTS or face/avatar.
- No secret reads/prints/sets.
- Do not ask Rob to rotate or set Anthropic secrets unless Rob explicitly chooses Anthropic later.
