# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Enable R2 on the backend (prerequisite to STT validation). Read-only discovery done. Reframed: R2 creds likely EXIST in creds.env but the stored endpoint was flagged malformed and the creds are not in the backend secret store. Rob to locate/fix/set; executor does NOT touch values.

## Done / verified
- STT bridge btg-stt READY (/healthz 200). Backend rev 00023 wired, /api/health 200, WS routing to /stream confirmed (101).
- Ellis 5 video sources file:// and unretrievable (failed upstream of STT).
- R2 NOT enabled in backend (cerebrium secrets list: no r2_ names).

## Discovery (read-only, RESULTS 23:24Z; no values printed)
- Exact backend config names (config.py:93-97, LOWERCASE, no env_prefix): r2_endpoint, r2_access_key, r2_secret_key, r2_bucket, r2_enabled(bool true).
- Env source: config.py:125-126 env_file=/opt/backtogether/.env; Cerebrium injects secrets as env. Set names exactly as lowercase field names (avoid case mismatch).
- STATE.md:199 — ~/btg-state/creds.env contains R2 keys (so creds were captured before).
- STATE.md:436 — "R2_ENDPOINT in creds.env malformed — not addressed." LIKELY ROOT CAUSE: malformed endpoint; R2 fails/stays disabled.
- STATE.md:43 / TURNOVER.md:49 claim "R2 live" — contradicts verified state (disabled). Doc vs behavior mismatch.

## Rob-only next steps
1. Locate R2 creds: Cloudflare dashboard R2 -> bucket name + account endpoint (https://<accountid>.r2.cloudflarestorage.com) + R2 API token (access key id + secret). OR recover from ~/btg-state/creds.env, FIXING the malformed endpoint. Do NOT paste values in chat.
2. Set the five lowercase names in Cerebrium project p-a907d7c5 (CLI form that worked earlier):
   cerebrium secret set r2_enabled "true"
   cerebrium secret set r2_endpoint "<endpoint>"
   cerebrium secret set r2_access_key "<id>"
   cerebrium secret set r2_secret_key "<secret>"
   cerebrium secret set r2_bucket "<bucket>"
   (If "secret set" errors on this CLI version, report the exact error; do not paste values.)
3. Verify names only: cerebrium secrets list | grep -i r2_  (executor re-verifies name-only).
4. Confirm to executor. Then executor proceeds: confirm-before-live redeploy -> /api/health -> verify R2 active read-only (no executor upload) -> docs -> STOP before upload/reprocess.

## DO NOT generate random secret values
Random/placeholder values make _r2_enabled True but R2 auth FAILS (worse than disabled). Use real Cloudflare R2 creds only.

## After R2 verified
- Re-upload ellis video or test persona [ROB-ONLY] -> reprocess [ROB-ONLY] -> executor verifies STT end-to-end (transcribed, segments>0, /stream logs). Then TTS (ROB-ONLY spend).

## ROB-ONLY (carried)
- Locate/fix/set R2 creds (values, dashboard/CLI). Upload/reprocess. Backend redeploy (confirm-before-live). TTS/face GPU. No secret values set by executor.

## Hard constraints
No secret values set/read/printed by executor. No prod DB writes/uploads by executor. No redeploy until 5 r2_ names confirmed present AND confirm-before-live. No random secret generation. No blind retry of file:// sources. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
