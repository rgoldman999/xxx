# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Deploy the /api/stt prefix fix and verify it resolves the /stream 404. Code APPLIED + COMMITTED (not deployed). Awaiting confirm-before-live deploy approval — btg-stt FIRST, then backend.

## Done (RESULTS 02:08Z)
- Applied 3 edits (prefix /api/stt on bridge + snapshot; backend _ws_url_from -> /api/stt/stream). Verified greps + diff; only 3 files changed.
- Committed: backtogether repo, branch main, hash d30301e. Tree clean. NOT pushed (commit only, per scope).
- Net: ws route /stream -> /api/stt/stream; bare /healthz (Cerebrium healthcheck) unchanged.

## Next step — ROB-ONLY confirm-before-live, IN ORDER
1. Redeploy btg-stt FIRST (snapshot has the prefix; new route /api/stt/stream must exist before the client points at it).
   - cmd: cd ops/cerebrium/stt && cerebrium deploy --config-file ./cerebrium.toml (logged bg)
   - smoke: /healthz 200; WS probe to /api/stt/stream -> dashboard Runs WS-close status (NOT GET 404).
   - HALT if /api/stt/stream still logs GET 404 -> path-depth disproven; do not deploy backend; test min_replicas=1 or escalate.
2. THEN redeploy backend (client now builds /api/stt/stream).
   - smoke: /api/health 200.
   - Order rationale: stt-first avoids a transient window where backend calls /api/stt/stream on an stt that still only serves /stream.
3. After both live (AUTO-RUN verify): real reprocess of a fresh video [ROB-ONLY trigger] -> audio_extracted -> transcribed, source_speaker_segments>0, dashboard /api/stt/stream WS activity.

## ROB-ONLY (carried)
- Approve each redeploy (confirm-before-live, separately); push (if wanted); upload/reprocess; min_replicas/spend; TTS/face. No secret values read/set by executor.

## Hard constraints
No deploy without explicit confirm-before-live (btg-stt and backend separately). No push unless Rob asks. No retry of 294c44ea until /api/stt/stream non-404. No upload/reprocess by executor. No TTS/face.

## Follow-on (NOT now)
- TTS/avatar bridges already use nested ws paths (/tts/stream, /avatar/stream) — only one level deep. If /api/stt/stream works but a one-level path wouldn't, may need /api/tts/stream etc. Re-evaluate at TTS bring-up. Flagged, not acted on.
- Small fix (draft later): STT bridge call attach a message so error string never blank.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).