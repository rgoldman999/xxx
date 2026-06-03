# CURRENT GATE

## Objective
Gap #1 D-2 internal Qwen routing verification — CLOSED (with formatter caveat). Pick the next gate.

## D-2 result — CLOSED (verified with formatter caveat)
Direct API/WS test executed by the USER on 2026-06-03.
- call_id = e1f9cdd4-6abc-4469-81b7-4d69637fc6fa
- POST /api/call/start -> 200 OK; created the FIRST prod calls row for user_id=c447365d-9201-4ebe-9be1-fe3a8e78411f (persona c40776dd "ellis", mode=audio, status=active, started 2026-06-03 20:58:10Z).
- WebSocket connected: "/api/call/ws/e1f9cdd4-6abc-4469-81b7-4d69637fc6fa" [accepted].
- llm_provider_route_decision log line EMITTED for that call (fires at call.py:582 on WS connect, immediately after resolve_provider_for_user at :576).

### Formatter caveat
Python logging.basicConfig (main.py:2) uses the default formatter, which DROPS extra={} fields. So the literal `llm_provider_override="qwen"` / `qwen_allowlist_match=true` fields were NOT visible in the log text — the line printed as bare `INFO:app.routers.call:llm_provider_route_decision`. The field values are therefore not greppable from logs without a formatter change.

### Why routing is still verified (chain, not log text)
- calls row user_id = c447365d... (verified in prod DB).
- route-decision code fired for that exact call_id (WS-accept line + route-log line, same call).
- deployed resolver/allowlist returns "qwen" for c447365d... (unit-proven + live-confirmed against the live INTERNAL_QWEN_USER_IDS value this session).
- no non-allowlisted calls exist: calls-by-non-c4473 = 0 (only this one call exists in prod).

## Known downstream (does NOT affect D-2)
After the route-log, the WS handler errored at call.py:629 on websockets.connect(...) to the TTS bridge http://localhost:8002 (start_call log: "TTS worker pinned ... base=http://localhost:8002") — bridge absent, connection refused. Route-log fires BEFORE this, so D-2 is unaffected. Confirms bridge tier still absent.

## Open item — active test call
calls row e1f9cdd4... is status=active, ended_at=None. Should be ended (POST /api/call/<id>/end) if not already. Executor does NOT place or end calls — user action.

## Remaining gap for fully usable calls
Absent bridge tier: STT (ws://localhost:8003), TTS (localhost:8002), face/avatar (localhost:8000). No STT/TTS/face Cerebrium app exists; no bridge/RunPod secrets set. Callable persona (real voice_id) needs STT + TTS minimum.

## Next gate — choose ONE
1. Logging formatter fix: make llm_provider_route_decision fields (user_id, llm_provider_override, qwen_allowlist_match) greppable in log text. Small backend code change (formatter / structured logging) + redeploy. Would let the literal override=qwen be read directly. NEW SCOPE — code + deploy, requires approval.
2. Bridge-tier bring-up plan: STT first, then TTS, face/avatar later. Substantial infra phase (deploy apps + set *_SERVICE_URL / RUNPOD_API_KEY secrets + smoke tests + persona retry). Path to a genuinely callable persona and a fully working call. NEW SCOPE — deploy + secrets, requires approval.

## Hard constraints (unchanged)
No deploy. No code changes. No env/secret changes. No bridge work. No prod DB writes except an explicitly approved user-run test. No calls placed OR ended by executor. Show diff and hold before any commit.

## User action needed
Pick next gate (1 or 2). End the active test call e1f9cdd4... if desired (user-run).
