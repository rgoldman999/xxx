# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Both fixable hypotheses (path-depth, scale-to-zero) are FALSIFIED by deploy tests. The remaining verified difference between our working WS app and the broken one is CPU vs GPU runtime — a Cerebrium platform question. Next: escalate to Cerebrium support. STT blocked end-to-end.

## Done / verified
- /api/stt prefix fix deployed -> /api/stt/stream STILL "GET 404". PATH-DEPTH FALSIFIED.
- btg-stt min_replicas=1 (always-on) test -> /api/stt/stream STILL "GET 404" (19:20:42). SCALE-TO-ZERO FALSIFIED. Reverted to min_replicas=0, redeployed (build-ce08b0a7). toml back to committed baseline (git clean). Always-on window ~6 min.
- disable_auth: ELIMINATED — BOTH apps have disable_auth=true (btg-stt toml:17, backend toml:7). Not the differentiator.
- Router-vs-app mounting: already eliminated (backend uses @router+include_router, WS works).

## Verified remaining differentiator
- WORKS: backtogether-backend — compute=CPU, min_replicas=1, disable_auth=true. /api/call/ws/{id} WS reaches container (Runs 1006).
- FAILS: btg-stt — compute=ADA_L4 (GPU), tested min_replicas 0 AND 1, disable_auth=true. /stream AND /api/stt/stream both "GET 404"; /healthz 200.
- Only remaining difference: CPU (works) vs GPU/ADA_L4 (fails) runtime. This is Cerebrium platform-internal -> escalate.

## Next step — ROB-ONLY: Cerebrium support escalation (plan C leg B)
- Send Cerebrium the evidence pack (RESULTS 02:28Z). Core question: why does a wss:// upgrade to a custom-runtime GPU app's @app.websocket route get routed/logged as a plain GET and 404'd to the container, while the identical pattern on our CPU app works? Is WS upgrade proxying unsupported/needs-a-flag on GPU custom runtimes?
- Optional cheap test if Rob wants before/with support (NOT auto): does WS work on our OTHER GPU app (btg-llm-qwen-4b, AMPERE_A10) if it has any ws route? If no GPU app anywhere proxies WS, strongly implicates GPU runtime. (Read-only-ish; only if a ws route exists there.)

## Code state
- /api/stt prefix fix committed (d30301e), ineffective alone, LEFT in place (harmless; reconcile path once real fix known).
- toml reverted to min_replicas=0 (matches committed baseline; uncommitted net-zero diff = none).
- Backend NOT deployed (still builds old /stream path; moot until a working fix exists).

## ROB-ONLY (carried)
- Cerebrium escalation; any further min_replicas/spend test; backend deploy; upload/reprocess; transport/arch decision; TTS/face. No secret values read/set by executor.

## Hard constraints
No further deploy/spend test without explicit approval. No backend deploy / reprocess until a path returns non-404 (WS reaches container). No push unless asked. No TTS/face.

## Architectural fallback (if Cerebrium confirms GPU custom-runtime WS unsupported)
- Move STT off streaming WS: e.g. batch/chunked HTTP transcribe endpoint on the GPU app (Cerebrium docs: Streaming + REST endpoints), with the backend posting audio and receiving transcript. Larger change to bridge + client. DECISION for Rob/Jeannine.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).