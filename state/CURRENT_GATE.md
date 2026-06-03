# CURRENT GATE

## Objective
Close Gap #1 D-2 internal Qwen routing verification.

## Current status
Qwen4B endpoint is live.
Backend routing seam is deployed.
INTERNAL_QWEN_USER_IDS allowlist is live for c447365d-9201-4ebe-9be1-fe3a8e78411f.
Frontend WS fix is live.
ffmpeg is fixed.
Callable persona remains blocked by absent STT/TTS/face bridge tier.

## Current blocker
UI cannot start a call because all personas have voice_id=None.
D-2 can still be closed through a direct authenticated API/WS test because backend start_call does not require persona readiness and route logging fires at WS connect.

## Safe next action
Run the direct API/WS D-2 test manually, then verify read-only:
- calls row exists for c4473…
- llm_provider_route_decision emitted
- resolver confirms c4473… → qwen
- no non-allowlisted user routed to Qwen

## Hard constraints
No deploy.
No code changes.
No env/secret changes.
No bridge work.
No prod DB writes except the user-run test creating one call row.
No calls placed by executor.

## User action needed
Run the direct API/WS test locally using browser session token, then report call_id and timestamp.

## Executor action after user test
Verify read-only:
- calls row exists for c4473…
- backend log emitted llm_provider_route_decision
- allowlist/resolver confirms c4473… routes to Qwen
- no non-allowlisted Qwen routing
