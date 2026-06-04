# NEXT REPLY

status: PENDING
updated_at: 2026-06-04T02:05:00Z
consumed_at:
consumed_by:
gate_commit: 5b819670ac9ef589c0535317f151a4b8eb320adf
classification_hint: AUTO-RUN draft code plan; deploy remains confirm-before-live

## body
Rob wants a way past the Cerebrium WebSocket blocker without waiting for support. Web research found a better bypass: move STT bridge invocation from WebSocket streaming to a plain HTTP transcription endpoint.

Sources found:
- Cerebrium REST endpoints are authenticated POST calls to app functions at /v4/<project>/<app>/<function>.
- Cerebrium async requests support ?async=true and can run up to 12 hours if response_grace_period is configured high enough.
- Cerebrium has an official Transcribe 1 hour podcast example that accepts base64 audio or a file_url and returns Whisper transcription results over HTTP, not WebSockets.
- Cerebrium also supports SSE streaming via generators, but for this pipeline batch HTTP is simpler and safer than SSE.

Recommendation:
Do not block on WebSocket proxying. Draft a non-WS STT path:
1. Add an HTTP endpoint/function on btg-stt, for example POST /transcribe or function transcribe_audio, accepting one of:
   - signed/public URL to the R2 audio object, or
   - r2 bucket/key plus R2 env access on the STT app, or
   - base64 audio only for small test files, not 35-minute production video.
2. Have backend stt_bridge.py call the HTTP endpoint with requests/httpx instead of websockets.connect for this batch transcription path.
3. Keep /stream untouched for later; add fallback or switch for STT batch only.
4. For 35-minute audio, prefer Cerebrium async=true or configure response_grace_period high enough. Backend should poll/result-fetch or receive webhook if available; for first validation, synchronous HTTP may be acceptable only if timeout is safely configured.
5. Add better error_message capture so STT failures include exception type/message and never remain blank.

Next executor action:
- Read-only inspect current btg-stt and backend STT code.
- Draft the minimal non-WS HTTP transcription patch and show diff.
- Include a small test plan.
- Do not deploy until confirm-before-live.
- Do not upload/reprocess until the HTTP path is deployed and smoked.

Hard constraints:
- no deploy without explicit approval
- no upload/reprocess/retry by executor
- no TTS or face/avatar
- no secret reads/prints/sets
- do not remove the existing /stream route; bypass it for batch STT
