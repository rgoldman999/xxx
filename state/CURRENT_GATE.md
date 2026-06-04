# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Diagnosis converged: btg-stt's gateway does NOT proxy WS upgrades for ANY path (app-wide), while our CPU backend's does. Path-depth, scale-to-zero, disable_auth, router-mounting all eliminated. Next: Cerebrium escalation (sharpened) and/or architectural transport change. STT blocked end-to-end.

## Proven (verified)
- btg-stt: wss upgrade to real route /api/stt/stream AND to a bogus path BOTH log "GET 404" (19:31 bogus, 19:20 real). Gateway forwards every path as plain GET -> no @websocket route matches -> 404. App-wide WS non-proxying.
- backend (CPU): real ws route /api/call/ws/{id} -> 1006 (WS reaches container); bogus -> 403. Gateway negotiates WS. WS WORKS.
- Eliminated by tests: path-depth (prefix fix 404), scale-to-zero (min_replicas=1 404, reverted), disable_auth (both apps true), router-vs-app mount (backend same pattern works).
- NOT proven: GPU-runtime as the mechanism — qwen GPU app is vLLM HTTP-only (no ws route), so no GPU+ws control. GPU-vs-CPU is n=1 each side.

## Code/infra state
- /api/stt prefix fix committed (d30301e), ineffective alone, LEFT in place.
- btg-stt min_replicas back to 0 (toml = committed baseline; deployed build-ce08b0a7). Bogus probe cold-started it; scales to 0 on cooldown. No lingering spend.
- backend NOT deployed.

## Next step — ROB DECISION
A. ESCALATE to Cerebrium (sharpened evidence in RESULTS 02:33Z): why does a GPU custom-runtime app (btg-stt, disable_auth=true) route ALL wss upgrades as plain GET->404 while our CPU app with the identical FastAPI pattern negotiates WS? Is GPU custom-runtime WS unsupported or flag-gated?
B. ARCHITECTURAL transport change (does not wait on Cerebrium): move STT off streaming WS. Options to scope:
   - Add a non-WS HTTP transcribe endpoint on btg-stt (POST audio -> JSON transcript); backend posts the extracted wav and gets text. Loses streaming/partial, fine for upload-time batch transcription (the failing path is upload reprocess of a finite file, NOT realtime). LARGEST leverage: upload-time STT does not need streaming at all.
   - Realtime call path would still want WS later, but that is a separate (also-blocked) concern; upload transcription can ship via HTTP POST.
   This is a DECISION (Rob/Jeannine) + a code change (bridge adds HTTP route; backend client adds HTTP call) = AUTO-RUN draftable, gated deploy.

## Recommendation (for Rob to weigh, not auto)
- Upload-time transcription (the currently-blocked path) does NOT need WebSockets — it is batch transcription of a finite extracted wav. A simple HTTP POST transcribe endpoint on btg-stt sidesteps the WS gateway issue entirely and unblocks STT validation now, without waiting on Cerebrium. Realtime-call WS can be solved separately later.

## ROB-ONLY (carried)
- Cerebrium escalation; approve any transport/code change (show-diff) + deploy (confirm-before-live); upload/reprocess; spend; TTS/face. No secret values read/set by executor.

## Hard constraints
No code change without show-diff + approval. No deploy without confirm-before-live. No backend deploy/reprocess until a working transcribe path exists. No further spend test without approval. No push unless asked. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).