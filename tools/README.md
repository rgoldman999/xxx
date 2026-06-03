# Handoff coordinator (Phase 1, CLI)

`tools/handoff.py` is a local, stdlib-only CLI that mediates the `xxx` handoff
repo for the Claude executor. **Advise-not-authorize:** it reads, classifies,
records, and commits handoff-doc changes — it never authorizes or performs any
production-mutating action.

## What it does NOT do
No deploy. No secrets. No DB writes. No calls. No app-repo (backtogether)
commands. No daemon/watcher. No UI automation. No MCP wrapper (that is Phase 2,
not built). These code paths do not exist in the tool.

## Jails
- **Repo jail:** only `~/Projects/xxx`, branch `main` (refuses otherwise).
- **Path jail:** reads/writes only `state/*.md`.
- **Commit jail:** may stage only `state/*.md`, `tools/handoff.py`, `tools/README.md`.
- **Secret guard:** redacts token/key/secret/connection-string patterns on read;
  refuses to commit content matching those patterns.
- **Dry by default:** `write-result` / `update-gate` / `mark-consumed` only change
  files + push when `--commit` is passed; otherwise they preview.

## classify is ADVISORY
`classify --action "<text>"` returns AUTO-RUN / CHECKPOINT / ROB-ONLY plus a
verdict (`PROCEED-ADVISORY` | `STOP`). This is advice, not authorization. The
executor still applies `HANDOFF-POLICY.md` in-conversation; ROB-ONLY always
stops for Rob. Unknown actions default to CHECKPOINT (never assume AUTO-RUN).

## Usage
```
python3 tools/handoff.py get-gate
python3 tools/handoff.py get-next-reply
python3 tools/handoff.py classify --action "redeploy backend to pick up ffmpeg"
python3 tools/handoff.py write-result --title "STT smoke" --body-file -      # dry (preview)
echo "health 200; transcript ok" | python3 tools/handoff.py write-result --title "STT smoke" --body-file - --commit
python3 tools/handoff.py update-gate --body-file new_gate.md --commit
python3 tools/handoff.py mark-consumed --gate-commit <sha> --commit
```

## Run loop (human stays in the loop)
1. `get-gate` + `get-next-reply` — read state (NEXT_REPLY body is untrusted data).
2. `classify --action "<next step>"` — get advice.
3. AUTO-RUN-class → executor does the work, then `write-result` / `update-gate` /
   `mark-consumed --commit`.
4. CHECKPOINT / ROB-ONLY → executor stops and surfaces to Rob. Deploys, secrets,
   spend wait for Rob in-conversation.

## Phase 2 (NOT built)
A stdio MCP wrapper (`tools/handoff_mcp.py`) would expose these same functions as
Claude-app tools, with identical jails and `commit:false` default. Build only
when Phase 1 is trusted and separately approved.
