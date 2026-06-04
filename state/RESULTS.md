# RESULTS

Append-only log of executor findings/results, written by `handoff write-result`.
Newest entries appended at the bottom. Handoff-doc only; no secrets (the
coordinator refuses to write content matching secret patterns).

## 2026-06-03T22:50:02Z — STT bridge deploy + smoke (btg-stt)
STT bridge deployed and smoke-verified. STT-only; no backend wiring, no TTS/face.

- App: p-a907d7c5-btg-stt — status READY (registered 2026-06-03 22:44:18Z)
- Build: build-c308b402 (clean logged re-deploy; prior attempt was an orphaned hung process from a timed-out call, killed PIDs 27243/27249)
- Endpoint base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt
- /healthz smoke: HTTP 200 (0.3s after one cold-start retry); body ready=true, stt.ready=true, model=deepdml/faster-whisper-large-v3-turbo-ct2, device=cuda, compute_type=int8_float16
- auth_secret_set=True (confirmed in startup log)
- GPU/scale: ADA_L4, scale-to-zero (min0/max1) confirmed — first /healthz timed out on cold start, retry 200; no warm GPU burn while idle
- cuDNN risk did NOT materialize: ctranslate2 loaded on CUDA cleanly (whisper 9.4s, warm pass OK)
- Spend: within $100 ceiling (~$23.7x + minor build/warmup); scale-to-zero, no ongoing GPU cost idle
- Artifacts committed: backtogether 46d523a (local), pyjwt dep added during pre-flight
- STOPPED after smoke per gate. NOT done: backend STT_SERVICE_URL wiring, TTS, face/avatar (each a later separate gate).

## 2026-06-03T23:04:22Z — Backend wired to STT bridge + redeployed (rev 00023); reprocess pending
Backend STT wiring complete and verified. STT path ready to validate via reprocess (Rob action). No TTS, no face/avatar.

DONE + VERIFIED:
- Backend stt service url setting confirmed present in backend secret store (name-only; Rob set it project-scoped). Bridge auth shared value also present.
- WS routing pre-verified: handshake to btg-stt /stream returned HTTP 101 Switching Protocols (Cerebrium/envoy routes WS to bridge 8003 /stream). Backend-constructed wss URL will reach the bridge.
- Backend redeployed: app backtogether-backend revision 00023, build build-c706e76b. Clean startup (uvicorn :8000, worker started). Logged background deploy (no orphan).
- Backend health: GET /api/health -> HTTP 200, {"status":"ok","service":"backtogether"}.

VERIFIED BEFORE-STATE (ellis persona c40776dd, read-only DB):
- 13 persona_sources: 5 video, 8 photo.
- All 5 video rows: processing_stage=audio_extracted, error_message=[Errno 111] Connection refused (the pre-wiring STT failure).
- photo rows: stage=None (no STT).
- source_speaker_segments (ellis): 0.
- video source_ids: 1154ccfa-1bb4-4a2a-8e27-62bc3665eb95, 368930f7-fdf0-411b-93ee-76a6141c78b9, 7fa2e508-aea4-4ecf-ad1d-abe19eddd3e6, d9caf45c-81bb-4630-a92d-febf527938a9, f149bd58-a66f-462f-a51c-10fe50614783.

CHECKPOINT (why stopped): reprocess endpoint POST /api/upload/_reprocess/{source_id} (a) mutates prod DB (resets source row), and (b) requires get_current_user owner auth. Both are ROB-ONLY per rule 5 (prod-data mutation; authenticated request as Rob). Executor will NOT call it. Rob triggers; executor then verifies read-only.

NEXT (Rob action): POST /api/upload/_reprocess/{source_id} for each of the 5 video ids, authed as Rob, against backend base https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend. Then executor verifies (AUTO-RUN, read-only): video rows advance audio_extracted -> transcribed; source_speaker_segments > 0; STT bridge logs show transcription requests.

NOT done: reprocess (Rob), TTS bridge, face/avatar. Spend within ceiling.

## 2026-06-03T23:12:26Z — Reprocess FAILED upstream of STT — source files are file:// local, not in R2 (new blocker)
Rob triggered reprocess for all 5 ellis video sources. Verified read-only: all 5 FAILED, but with a NEW error upstream of STT — not the prior errno-111. STT wiring is NOT implicated.

OBSERVED (read-only DB, db now 23:09:19Z; rows updated ~23:08:33-35Z):
- All 5 video sources: processing_status=failed, processing_stage=None, transcript=False, speech_seconds=0, speakers=0, audio_duration=0.
- source_speaker_segments (ellis): 0 (unchanged).
- error_message on each = the bare source file path (the FileNotFoundError arg).

ROOT CAUSE (traced to source, not assumed):
- processing.py:149-161 resolves the source file: tries ostore.get(storage_uri) [R2 download], else local file_path, else raises FileNotFoundError(file_path) at line 156-157.
- The 5 sources have storage_uri = file:///opt/backtogether/uploads/<persona>/<uuid>.mov  (LOCAL disk path, scheme file://), NOT an r2:// URI.
- They were uploaded to a previous deployment's local container disk and never pushed to R2. The current backend (rev 00023, fresh container) has no such local file, and ostore cannot fetch a file:// URI from R2.
- => Pipeline fails at source-file open, BEFORE audio extraction / STT.

STT NOT REACHED (so STT wiring remains unvalidated, but also un-disproven):
- btg-stt logs since reprocess: no /stream WS / transcription requests observed (consistent with upstream failure; note cerebrium logs is sometimes flaky).

CLASSIFICATION: CHECKPOINT / new blocker. NOT a blind-retry case — retrying reprocess will hit the same FileNotFoundError because the source files do not exist in retrievable storage.

NEXT (decision for Rob — options, NOT yet chosen):
- A. Re-upload the 5 source videos for ellis (fresh upload pushes to R2 with an r2:// storage_uri), then reprocess. This is the clean path to actually exercise STT.
- B. Investigate whether the original local files still exist on any persistent volume and can be backfilled to R2 (uncertain; prior container disks may be gone).
- C. Test STT with a brand-new test persona + a fresh small video upload (smallest path to validate STT end-to-end without depending on the orphaned ellis sources).
NOTE: uploads / re-uploads are Rob actions (auth + prod data). Executor will verify read-only after.

NOT done: STT end-to-end still unvalidated; no TTS; no face/avatar. No retry performed.

## 2026-06-03T23:15:49Z — Option A blocked: R2 not configured in backend — uploads fall back to file:// (root cause of unretrievable sources)
Chose option A (re-upload ellis videos). Pre-flight read-only check found A will FAIL the same way until R2 object storage is configured. STOPPED before any upload.

ROOT CAUSE (traced to source, read-only):
- object_store.py:12 — writes go to R2 only if R2_ENABLED; otherwise fall back to file:// local disk.
- _r2_enabled() (object_store.py:36-38) requires ALL of: r2_enabled, r2_endpoint, r2_access_key, r2_secret_key, r2_bucket.
- config.py:93-97 defaults: r2_enabled=False; r2_endpoint/access_key/secret_key/bucket all empty.
- Backend Cerebrium secrets (names only) contain NONE of the R2 vars (no R2_* / bucket entries among the 20 secrets).
- => _r2_enabled() is False in the deployed backend. Every upload falls back to file:// local disk.

IMPLICATION:
- This is WHY the original ellis sources are file:// and unretrievable after redeploy — R2 was never enabled.
- A re-upload (option A) would produce another file:// URI on the fresh container and fail identically after any redeploy. Option A cannot work until R2 is configured.
- This is a storage-layer prerequisite, not an STT problem. Per stack docs R2 is the intended object store (process-and-delete, r2:// URIs); it is currently unconfigured.

NEW REQUIRED PREREQUISITE (ROB-ONLY): configure R2 on the backend before any upload-based STT validation:
- Set backend secrets: R2 endpoint, access key, secret key, bucket, and r2_enabled=true (exact env names per config.py: r2_endpoint/r2_access_key/r2_secret_key/r2_bucket/r2_enabled). Secret VALUES = Rob (never via chat). Per memory, R2 bucket + creds exist in Rob's creds files.
- Then backend redeploy to pick them up (confirm-before-live), and verify _r2_enabled via an upload that yields an r2:// storage_uri.

REVISED PATH:
1. (ROB-ONLY) Rob sets R2 backend secrets + r2_enabled=true.
2. (CHECKPOINT) backend redeploy (confirm-before-live).
3. (verify) read-only: confirm a fresh small upload yields storage_uri starting r2:// (not file://).
4. (ROB-ONLY) re-upload ellis videos (or a test persona video) -> reprocess.
5. (AUTO-RUN verify) sources advance past audio_extracted -> transcribed; source_speaker_segments > 0; btg-stt logs show /stream requests.

NOT done: no upload performed, no R2 secrets set, no redeploy, no retry. STT still unvalidated end-to-end. No TTS/face.

## 2026-06-03T23:20:53Z — HALT: R2 secrets reported set but NOT present in backend secret store — did not redeploy
Rob reported the five R2 names set. Read-only verification BEFORE redeploy found they are NOT in the Cerebrium secret store the backend reads. HALTED — did not redeploy (a redeploy now would still have R2 disabled and uploads would still fall back to file://).

OBSERVED (read-only, names only):
- cerebrium secrets list = 20 secrets, UNCHANGED count, and NONE of r2_enabled / r2_endpoint / r2_access_key / r2_secret_key / r2_bucket present.
- Full list: ADMIN_PIN, ANTHROPIC_API_KEY, APP_SECRET_KEY, BRIDGE_AUTH_SECRET, BT_AUTO_MIGRATE, BT_RUN_SCHEDULER, BT_RUN_WORKER, DATABASE_URL, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, INTERNAL_QWEN_USER_IDS, OPENAI_API_KEY, QWEN_API_TOKEN, QWEN_BASE_URL, REDIS_URL, SENDGRID_API_KEY, STRIPE_PUBLIC_KEY, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, STT_SERVICE_URL.

LIKELY CAUSES (Rob to disambiguate; not guessed-and-proceeded):
- Secrets set in a different project/scope than the CLI context (this CLI = the one showing backend secrets all session).
- Set in a different store: dashboard "Project Secrets" vs app "Secrets" tab vs env — backend reads Cerebrium project secrets.
- Set command silently failed (CLI version: earlier `cerebrium secret set NAME VALUE` worked; NEXT_REPLY suggested `secrets add name="value"` — syntax mismatch possible).

DID NOT: redeploy, set secrets, upload, retry. Precondition (5 R2 names present) NOT met, so the gate's step 1 is not satisfied.

NEXT (ROB-ONLY): Rob re-checks where/how the R2 secrets were set; confirm via `cerebrium secrets list` (his side) shows the five r2_ names in project p-a907d7c5. If his CLI shows them but this one does not, the two CLIs target different projects — fix that. Once the five names appear in `cerebrium secrets list` for p-a907d7c5, executor proceeds: confirm-before-live redeploy -> /api/health -> verify R2 active (read-only) -> docs -> stop.

## 2026-06-03T23:24:44Z — R2 read-only discovery: creds exist but endpoint flagged malformed; backend reads lowercase env from .env / project secrets
Followed NEXT_REPLY (AUTO-RUN read-only discovery). Key findings reframe the blocker: R2 creds likely EXIST but the endpoint was recorded malformed, and the stack doc claims R2 live while verified behavior says disabled. Executor did NOT read secret values, set secrets, or deploy.

EXACT BACKEND CONFIG NAMES (config.py:93-97, lowercase, no env_prefix):
- r2_endpoint
- r2_access_key
- r2_secret_key
- r2_bucket
- r2_enabled (bool; set true)
Env mapping: config.py:125-126 -> class Config env_file=/opt/backtogether/.env. Pydantic reads env (Cerebrium injects secrets as env). Field names are LOWERCASE; if secrets were set UPPERCASE (R2_ENDPOINT...), confirm pydantic case handling — safest is to set names exactly as the lowercase field names.

DISCOVERY (read-only, no values printed):
- STATE.md:199 — ~/btg-state/creds.env contains R2 keys (assembled 2026-05-06, mode 600). So R2 creds were captured previously; "no R2 secrets available" may mean not-in-Cerebrium, not non-existent.
- STATE.md:436 — "R2_ENDPOINT in creds.env malformed — not addressed this session." LIKELY ROOT CAUSE: the stored R2 endpoint is malformed; R2 would fail/stay disabled even if set. Needs correction.
- STATE.md:43 / TURNOVER.md:49 — stack docs claim "R2 live", but today's verification shows R2 NOT enabled in backend (no r2_ secrets present). Doc-intent vs verified-behavior mismatch.
- STATE.md:435 — note of 8 R2 source blobs (~8.6GB) referenced previously (suggests R2 was used at some point).

REFRAME: This is not "generate new R2 creds from scratch." It is: (a) locate existing R2 creds in creds.env, (b) FIX the malformed r2_endpoint, (c) set the five lowercase names into the backend's Cerebrium project secrets (p-a907d7c5), (d) redeploy. Generating random values would be harmful (non-empty -> _r2_enabled True but auth fails).

ROB-ONLY NEXT STEPS:
1. Locate R2 creds: Cloudflare dashboard R2 -> bucket name + account R2 endpoint (https://<accountid>.r2.cloudflarestorage.com) + an R2 API token (access key id + secret access key). Or recover from ~/btg-state/creds.env, FIXING the malformed endpoint.
2. Set the five names in Cerebrium project p-a907d7c5 (values never in chat). Use the CLI form that worked earlier:
   cerebrium secret set r2_enabled "true"
   cerebrium secret set r2_endpoint "<endpoint>"
   cerebrium secret set r2_access_key "<id>"
   cerebrium secret set r2_secret_key "<secret>"
   cerebrium secret set r2_bucket "<bucket>"
   (If "secret set" errors on this CLI version, report the exact error; do not paste values.)
3. Verify names only: cerebrium secrets list | grep -i r2_
4. Confirm to executor (name-only). Then executor: confirm-before-live redeploy -> /api/health -> verify R2 active read-only -> docs -> stop before upload/reprocess.

NOT done: no values read/printed, no secrets set, no deploy, no upload, no retry. STT still unvalidated.

## 2026-06-03T23:40:41Z — R2 secrets present; backend redeployed rev 00024; R2-active not fully verifiable read-only (diag route is owner-authed)
R2 prerequisite advanced. Five R2 names now present; backend redeployed; health green. R2 *config present* but R2 *working* not yet proven (requires owner-authed call or an upload = Rob). Stopped at the upload boundary.

VERIFIED (read-only, executor):
- Backend secrets now include all five: r2_access_key, r2_bucket, r2_enabled, r2_endpoint, r2_secret_key. Total 25 (was 20).
- Backend redeployed rev 00024 (logged bg deploy, no orphan), clean startup.
- GET /api/health -> HTTP 200 {"status":"ok","service":"backtogether"}.
- These secrets inject as env into rev 00024, so _r2_enabled() (object_store.py:36-38) should now be True (all five non-empty).

NOT YET VERIFIED (boundary):
- _r2_enabled() only checks non-empty; it does NOT validate the endpoint is well-formed or that creds authenticate. Given the prior 'malformed endpoint' flag, non-empty != working.
- R2-active confirmation requires either:
  (a) GET /api/upload/_diag — returns recent upload events incl storage_uri_val + r2_error — but it is owner-authed (get_current_user). Executor will not call as Rob.
  (b) an actual upload, then read storage_uri scheme.
  Both are Rob actions. No unauthenticated R2-status route exists.

NEXT (ROB-ONLY, the upload boundary):
1. Rob does ONE small test upload (any persona), then either:
   - GET {backend_base}/api/upload/_diag authed as Rob, and report storage_uri_val + r2_error for that event (these are not secret values — scheme + error field), OR
   - tell executor the source_id and executor reads storage_uri scheme read-only from DB.
2. PASS = storage_uri starts r2:// and r2_error null. FAIL (file:// or r2_error set) = endpoint/creds still wrong -> diagnose, do NOT proceed to ellis re-upload.
3. On PASS: re-upload ellis videos (or use the test upload's persona) -> reprocess -> executor verifies STT end-to-end (transcribed, segments>0, /stream logs).

NOT done: no upload by executor, no reprocess, no auth'd call as Rob, no TTS/face, no retry of file:// ellis sources.

## 2026-06-04T00:42:03Z — R2 verification FAILED: post-redeploy uploads still file:// — R2 not active despite secrets present
Rob uploaded test files after rev-00024 redeploy. Read-only DB check of newest sources: still file://. R2 is NOT persisting to R2. FAIL per gate. Did NOT proceed to ellis re-upload/reprocess.

OBSERVED (read-only DB, db now 2026-06-04 00:39:19Z; user c447365d):
- Newest video ad504088 (created 00:38:55, AFTER redeploy): storage_uri scheme = file://, audio_uri NULL, status=failed, error="ffmpeg error: ffmpeg version 5.1.9-0+deb12u1 ..." (a NEW, separate downstream error).
- Other file 15c8350b (created 00:37:44, after redeploy): completed, scheme = file://.
- => Uploads post-rev-00024 still land on file:// local disk. _r2_enabled() is effectively False OR R2 write fails and falls back to file://.

CONCLUSION: R2 config present (5 names, total 25 secrets) but R2 NOT working. Confirms the earlier caution: _r2_enabled() non-empty check != functioning R2. Likely causes (not disambiguated; values not read):
- r2_endpoint malformed (prior STATE.md:436 flag) -> client init/write fails -> file:// fallback.
- r2_enabled bool not parsed truthy (value not exactly "true"/expected).
- creds/auth failure in put_local_file -> silent file:// fallback.
- backend logs not retrievable (cerebrium logs flaky this session) to see the exact R2 error.

SECONDARY (note, not the blocker): newest video ad504088 failed with an ffmpeg error (different from file:///errno). Downstream; revisit after R2 fixed.

NEXT (ROB-ONLY diagnosis; executor will not read secret values):
1. Rob verifies the R2 secret VALUES locally (not in chat):
   - r2_endpoint is a well-formed URL: https://<accountid>.r2.cloudflarestorage.com (no trailing slash, no bucket in path, https scheme). The prior "malformed endpoint" note likely applies.
   - r2_enabled value is exactly the string true (lowercase) so the bool parses truthy.
   - r2_access_key / r2_secret_key are a valid R2 API token pair for r2_bucket.
2. Optionally Rob pulls backend logs (dashboard App Logs) for the upload at 00:38 to see the exact R2 error line (object_store fallback reason).
3. Re-set any wrong value (Rob), redeploy (confirm-before-live), re-test one upload, executor re-checks scheme read-only. PASS only when storage_uri starts r2://.

NOT done: no ellis re-upload, no reprocess, no secret values read by executor, no TTS/face, no blind retry.

## 2026-06-04T01:10:48Z — R2 values corrected; backend redeployed rev 00025; /api/health 200 — awaiting one fresh upload to verify r2://
R2 secrets re-set with corrected values (Rob); backend redeployed to pick them up; health green. Now need ONE fresh upload (Rob) to confirm storage_uri=r2:// before proceeding.

VERIFIED (read-only, executor):
- Five R2 names present (r2_access_key, r2_bucket, r2_enabled, r2_endpoint, r2_secret_key). (Re-set existing keys -> count stays 25; names confirmed.)
- Backend redeployed rev 00025 (logged bg deploy, no orphan); clean startup; startup-log GET /api/health 200.
- GET /api/health -> HTTP 200 {"status":"ok","service":"backtogether"}.

WHY A FRESH UPLOAD IS NEEDED:
- The two earlier file:// uploads (00:37-00:38) were against rev 00024 with the OLD (bad) R2 values. They cannot retroactively become r2://.
- Verification of the corrected R2 config requires a NEW upload made against rev 00025. _r2_enabled() non-empty check passes, but only an actual upload proves the endpoint/creds work (r2:// + no r2_error).

NEXT (ROB-ONLY, the upload boundary):
1. Rob performs ONE small fresh upload (any persona) against rev 00025.
2. Then either:
   - Rob hits GET {backend_base}/api/upload/_diag (authed) and reports storage_uri_val + r2_error for that event (scheme + error field only), OR
   - Rob gives executor the new source_id / persona; executor reads storage_uri scheme read-only from DB.
   backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
3. PASS = storage_uri starts r2:// AND r2_error null. FAIL = file:// or r2_error -> R2 still wrong, diagnose (do NOT proceed).
4. On PASS: re-upload ellis videos / use a good source -> reprocess [ROB-ONLY] -> executor verifies STT end-to-end (transcribed, segments>0, /stream logs).

NOT done: no executor upload/authed-call/reprocess, no TTS/face, no retry of file:// ellis sources. STT still unvalidated end-to-end (pending a durable r2:// source).

## 2026-06-04T01:13:23Z — R2 VERIFIED working: fresh upload lands r2:// (PASS)
R2 prerequisite COMPLETE. Fresh upload against rev 00025 persisted to R2. STT end-to-end now unblocked (needs a video source in R2 + reprocess = next gate).

VERIFIED (read-only DB, db now 2026-06-04 01:13:06Z):
- Newest upload 066286e3 (photo, created 01:12:26, AFTER rev-00025 redeploy): storage_uri scheme = r2://, status=completed, error none. => PASS.
- Clear before/after: all sources <=00:38 (rev 00024, bad R2 values) = file://; the 01:12 upload (rev 00025, corrected values) = r2://. R2 now active.
- The corrected r2_endpoint (and values) fixed it.

NOTE: the verifying upload was a PHOTO (no STT path). It proves R2 persistence. STT end-to-end still requires a VIDEO source that lands in r2:// then reprocess — that is the NEXT gate (ROB-ONLY), and the current gate said stop before ellis re-upload/reprocess. STOPPED here.

NEXT GATE (ROB-ONLY trigger, then AUTO-RUN verify):
1. Rob uploads a fresh VIDEO source (new persona or re-upload to ellis) — now lands r2://.
2. Rob triggers reprocess for that video source (POST /api/upload/_reprocess/{source_id}, authed).
3. (AUTO-RUN) executor verifies read-only: video advances audio_extracted -> transcribed; source_speaker_segments > 0; btg-stt logs show /stream transcription requests. This finally exercises the STT bridge end-to-end.
Do NOT reuse the old file:// ellis videos (they are not in R2; re-uploading fresh is required). Do NOT blind-retry them.

NOT done: no executor upload/reprocess, no TTS/face. STT bridge still not yet exercised end-to-end (no video has reached it).

## 2026-06-04T01:20:08Z — STT end-to-end attempt: r2:// video reached audio_extracted then FAILED at diarize/STT with EMPTY error — needs Rob log/inspect
First r2:// video reprocess got FURTHER than ever (extracted audio from r2:// source) then failed at the diarize/STT step with NO persisted error. Exact cause not obtainable read-only (bridge/backend logs not retrievable; richer detail is owner-authed). HALT, no blind retry.

PROGRESS (this is new — prior videos never got here):
- New video 294c44ea (persona 14578822), storage_uri=r2:// (durable). 
- Advanced past prior failure modes: reached stage=audio_extracted with audio_duration_seconds=2085.94 (audio extraction from r2:// SUCCEEDED; ~35 min track). No errno-111, no FileNotFound.

FAILURE (verified read-only DB, db 01:16:51Z):
- status=failed, stage=audio_extracted, error_message=EMPTY, transcript_len=0, speech_seconds=0, speakers=0, source_speaker_segments=0. updated 01:16:00.
- So it failed AFTER audio extraction, at the diarize/transcribe (STT bridge) step, but no error string was persisted.

CODE PATH (processing.py read-only):
- After audio_extracted: silence gate (skip if duration<30 OR mean_db<-45). duration=2085s passes the duration check; would only skip if audio < -45dB (then it would go to 'transcribed', not 'failed') — so silence gate is NOT the failure.
- Else diarize_and_transcribe -> stt_bridge.transcribe_file; on exception it RE-RAISES, and the comment says the outer process_source handler captures it into source.error_message.
- CONTRADICTION: row failed at this step but error_message is EMPTY. Either the outer handler isn't persisting the message on this path, or failure is elsewhere. DB cannot reveal which.

DIAGNOSIS BLOCKED read-only:
- btg-stt logs and backend logs not retrievable this session (cerebrium logs flaky — returns empty).
- Richer per-source fields are behind owner-authed GET /api/upload/_inspect/{source_id}.

NEXT (ROB-ONLY, to get the actual error — pick any):
1. Rob: GET {backend_base}/api/upload/_inspect/294c44ea-9784-42eb-988a-701a11d7c448 (authed) -> report processing fields / any error detail.
2. Rob: dashboard -> backtogether-backend App Logs around 01:15-01:16 -> find the diarize/STT exception (and btg-stt App Logs for whether a /stream request arrived).
3. Likely candidates to look for in logs (NOT assumed): stt_bridge could not fetch the r2:// audio (does the bridge have R2 access, or does the backend stream bytes to it?); WS auth mismatch; transcribe_file timeout on a 35-min file; or an exception before error_message is written.
Do NOT blind-retry 294c44ea until the error is known.

NOTE: console error Rob saw ("listener indicated async response... message channel closed") is a browser-EXTENSION message, unrelated to backend/STT.

NOT done: STT still not validated end-to-end (failed at STT step, cause unknown). No TTS/face. No retry.

## 2026-06-04T01:26:24Z — STT failure diagnosis (read-only): messageless exception at STT WS call; leading cause = timeout/cold-start on 35-min file. Two red herrings eliminated.
Diagnosed source 294c44ea read-only against processing.py / stt_bridge.py using the _inspect output. Root cause not 100% confirmable read-only (needs the exception TYPE from logs), but narrowed to a strong leading hypothesis; two misleading signals eliminated.

_INSPECT KEY FIELDS:
- processing_status=failed, processing_stage=audio_extracted, error_message="" (EMPTY).
- storage_uri=r2://backtogether-prod/.../a8d748b3....mp4 ; audio_storage_uri=r2://.../294c44ea....wav ; audio_duration_seconds=2085.94 (35 min).
- transcript_len=0, speech_seconds=null, embedding_rows=0, face_detected=true.
- env_stt_bridge_url_present=false ; env_hf_token_present=false.

ELIMINATED (red herrings, verified in source):
1. env_stt_bridge_url_present=false is MISLEADING. _inspect (upload.py:97) checks STT_BRIDGE_URL/BRIDGE_URL — legacy names we did NOT set. The actual STT client (stt_bridge.py:54 _base_url) reads settings.stt_service_url = STT_SERVICE_URL, which IS set+verified. So the bridge URL is configured correctly; the diagnostic checks the wrong var.
2. r2:// audio path is NOT passed to STT. processing.py: file_path resolved local via ostore.get (line 153), audio_path = local .wav (174), diarize_and_transcribe(audio_path) (216) gets the LOCAL path. transcribe_file receives a real local file, not r2://.

WHY error_message IS EMPTY (mechanism, verified):
- Outer handler processing.py:407-408 sets processing_status=failed; error_message=str(e)[:500]. The message is empty because str(e) is EMPTY -> the propagated exception has no message string (bare/messageless exception: e.g. timeout/CancelledError/connection-close raised with no args).

LEADING HYPOTHESIS (strong, NOT yet confirmed):
- Failure is at the STT WebSocket call (stt_bridge.transcribe_file -> WS /stream), on a 35-min (2085s) file.
- btg-stt is scale-to-zero (min_replicas=0). A reprocess WS connect can race the GPU cold-start (container spin-up + whisper load ~9s). websockets.connect open_timeout (seen =15s on the call path) could time out on connect during cold start -> messageless TimeoutError/connection exception -> str(e) empty. OR a long-file transcription timeout/disconnect mid-stream.
- Consistent with: empty error, failure right after audio_extracted, no transcript, no /stream visible (logs unretrievable so not dispositive).

NOT CONFIRMED because: exception TYPE is only in btg-stt/backend logs (cerebrium logs flaky/empty this session) or would need code instrumentation. Did NOT assert as fact; did NOT retry.

NEXT (to confirm + fix — options for Rob/planning, NOTHING done):
1. Rob: dashboard btg-stt App Logs + backend App Logs ~01:15-01:16 -> the exact exception at the WS call (timeout? connection refused? cold-start?). This disambiguates.
2. If cold-start race: options — pre-warm btg-stt (min_replicas=1 temporarily; ROB-ONLY spend change) for the test, OR increase WS open_timeout / add connect retry in stt_bridge.py (code change, AUTO-RUN draft+show-diff, confirm-before-live deploy).
3. If long-file timeout: chunk the audio or raise the per-file timeout (code change, same gating).
4. Separately worth a fix: the empty-error path — transcribe_file should attach a message (type name) so error_message is never blank (small AUTO-RUN code fix, show-diff).

Do NOT retry 294c44ea until the exception type is known. No TTS/face. No deploy/upload/reprocess by executor.

## 2026-06-04T01:28:49Z — NEXT_REPLY (01:25) SUPERSEDED: its STT_BRIDGE_URL premise is a red herring already disproven in source (RESULTS 01:26). Current gate d35f137 stands.
get_next_reply returned an instruction (updated 01:25) authored from the raw _inspect output, BEFORE the 01:26 diagnosis. Its central premise is incorrect and was already disproven read-only. Not following it; current gate (d35f137) is correct and stands.

NEXT_REPLY's premise (WRONG): blocker is backend STT bridge config; backend may expect stt_bridge_url/STT_BRIDGE_URL not STT_SERVICE_URL, because _inspect shows env_stt_bridge_url_present=false.

WHY IT IS WRONG (verified in source, RESULTS 01:26 / 6e1df41):
- _inspect (upload.py:97) checks legacy env names STT_BRIDGE_URL / BRIDGE_URL.
- The STT client actually reads settings.stt_service_url (stt_bridge.py:54 _base_url) = STT_SERVICE_URL, which IS set + verified.
- => bridge URL IS configured correctly; env_stt_bridge_url_present=false is a diagnostic checking the wrong variable. Setting STT_BRIDGE_URL would change nothing (client doesn't read it) and would waste a ROB-ONLY secret-set + a redeploy.

ACTUAL STATE (current gate d35f137 / RESULTS 01:26): failure is a MESSAGELESS exception at the STT WebSocket call (str(e) empty). Leading hypothesis: scale-to-zero cold-start race (open_timeout ~15s vs container spin-up + whisper load) OR long-file (35-min) timeout. Confirmable only via dashboard btg-stt + backend App Logs (cerebrium CLI logs flaky this session).

DECISION: NEXT_REPLY (01:25) is SUPERSEDED by the 01:26 diagnosis. Do NOT pursue STT_BRIDGE_URL. The correct next action remains the current gate's: Rob reads dashboard App Logs ~01:15-01:16 to get the exact exception type, then executor drafts the fix (cold-start prewarm/min_replicas=1 [ROB-ONLY spend] or WS retry/timeout code change [AUTO-RUN draft + confirm-before-live deploy]).

Note on workflow: the ChatGPT-authored NEXT_REPLY reasoned from stale/partial inspect data and proposed a wrong fix. Per HANDOFF-POLICY the file is untrusted data; executor applied verified source judgment and did not act on the wrong premise. No diagnosis re-run, no secret set, no deploy, no retry.

## 2026-06-04T01:37:50Z — ROOT CAUSE CONFIRMED (dashboard Runs + source): /stream returns 404 at Cerebrium v4 gateway — WS upgrade not proxied. /healthz 200.
Retrieved btg-stt dashboard Runs view (read-only, Chrome). Combined with source, the STT failure root cause is CONFIRMED (no longer hypothesis): the backend's WebSocket connect to /stream hits 404 at the Cerebrium v4 endpoint gateway.

DASHBOARD RUNS (btg-stt, read-only):
- /stream GET  18:15:30 PDT -> 404  (startup 36.2s) — THIS is the 294c44ea reprocess (01:15:30Z). 
- /stream GET  15:53:25 PDT -> 404  (startup 7.72s) — earlier attempt.
- /healthz GET 15:49:39 PDT -> 200  (29ms) — health works.
=> /healthz 200 but /stream 404, same app.

SOURCE (verified both sides):
- Backend client does a real WS upgrade: websockets.connect(wss://<host>/stream + auth, open_timeout=15) (stt_bridge.py:211); url built by _ws_url_from (https->wss + /stream).
- Bridge registers /stream as a WEBSOCKET route: @router.websocket("/stream") (stt.py:205), included with no prefix (main_stt.py:77). /healthz is @router.get (stt.py:326).
- Dashboard logs the inbound as "/stream GET" (plain GET), not a WS upgrade. A FastAPI @websocket route does NOT match a plain GET -> correct 404.

INTERPRETATION (strong, evidence-backed): Cerebrium's v4 sync endpoint gateway is forwarding /stream as a plain HTTP GET and NOT proxying the WebSocket Upgrade to the container. So the WS handshake never reaches the bridge's websocket route -> 404 -> backend websockets.connect raises (messageless) -> processing.py:407 sets error_message=str(e)="" -> source fails at audio_extracted with empty error. ALL prior symptoms now explained.
- Reconciles earlier 101: the direct raw probe hit the container with upgrade headers intact (101); the real backend calls traverse the v4 gateway, which does not upgrade -> 404.

OPEN QUESTION (cannot resolve from repo; NOT guessing): does Cerebrium's v4 sync REST endpoint support WebSocket-upgrade proxying at all, or does WS require a different endpoint/path/invocation (e.g. a different host, a /ws path, the run-async/websocket-specific URL, or persistent-connection config)? This is Cerebrium platform behavior -> needs Cerebrium docs, not a code change on inference.

NEXT (to determine the FIX — read-only research + Rob, NOTHING changed):
1. Check Cerebrium docs for WebSocket support on v4 apps: is wss to api.aws...v4/p-.../btg-stt/stream supported, or is there a dedicated WS endpoint/host/path? (executor can web-search Cerebrium docs read-only.)
2. Possible outcomes:
   a. WS supported but different URL shape -> fix _ws_url_from / STT_SERVICE_URL base (code or secret; gated).
   b. v4 sync endpoint does NOT proxy WS -> architectural: the STT bridge's streaming /stream design may be incompatible with Cerebrium v4 request/response apps; would need a non-WS transcribe path OR a different transport/host. This is a DECISION (Rob/Jeannine), not an executor fix.
3. Do NOT change code, redeploy, or retry until the Cerebrium WS support question is answered.

NOT done: no code change, no deploy, no retry, no TTS/face. Root cause confirmed; fix pending Cerebrium WS capability check.

## 2026-06-04T01:44:11Z — Cerebrium docs check: v4 WS IS supported on custom runtime; our URL shape + entrypoint are correct. 404 cause narrowed to gateway upgrade not reaching container.
Read-only Cerebrium docs/web research complete. WebSockets ARE supported on Cerebrium v4 (custom runtime). Our btg-stt config, URL shape, and uvicorn bind are all correct per docs — so this is NOT platform-incompatible and NOT a simple URL typo. The 404 is the WS upgrade arriving as a plain GET at the container.

SOURCES (read-only):
- Cerebrium WebSocket docs: https://cerebrium.ai/docs/endpoints/websockets (also docs.cerebrium.ai/cerebrium/endpoints/websockets)
- Twilio voice agent example (FastAPI @app.websocket("/ws") on Cerebrium): https://docs.cerebrium.ai/v4/examples/twilio-voice-agent
- Multilingual STT voice-agent blog (faster-whisper + WS on Cerebrium): https://cerebrium.ai/blog/overcoming-transcription-challenges-for-multilingual-ai-voice-agents
- Realtime coding assistant (wss URL shape): https://cerebrium.ai/blog/building-a-real-time-coding-assistant
- Landing page feature list incl "WebSocket endpoints" + "Streaming endpoints": https://cerebrium.ai/

ANSWERS:
1. Do v4 apps support inbound WebSockets? YES. Documented first-class. REQUIRES a custom runtime ([cerebrium.runtime.custom] with port + entrypoint + healthcheck_endpoint + readycheck_endpoint).
2. Correct URL shape: wss://api.aws.<region>.cerebrium.ai/v4/<project-id>/<app-name>/<websocket-function-name>, where <websocket-function-name> = the FastAPI @app.websocket("<path>") route path. For us: wss://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt/stream  (route is @router.websocket("/stream")). Our backend _ws_url_from builds exactly this -> URL shape is CORRECT.
3. Limits/timeouts: docs page itself lists no explicit WS message-size/duration cap; healthcheck_endpoint non-200 => restart, readycheck_endpoint non-200 => removed from routing. (Long 35-min streams: no documented hard cap found; scale-to-zero cold start adds first-call latency. A dedicated WS timeout/limit doc was not found in this pass.)
4. If WS not supported: N/A — it IS supported. (Alternative transports Cerebrium documents: Streaming endpoints (HTTP chunked) and REST; relevant only if we later choose to move off WS.)
5. CLASSIFY NEXT PATH: NOT "platform incompatible"; NOT "URL/path fix only" (URL already correct). Best-supported classification = CONFIG/RUNTIME issue on the deployed app: the WS upgrade is not reaching the container (dashboard logged inbound as "/stream GET" 404), despite a correct custom-runtime toml and correct uvicorn bind.

ELIMINATED (docs + source, read-only):
- WS-unsupported: NO — supported on custom runtime.
- Wrong URL shape: NO — matches docs pattern (.../btg-stt/stream).
- Bad entrypoint/bind: NO — main_stt.py runs uvicorn.run("main_stt:app", host=0.0.0.0, port=8003); /healthz 200 proves uvicorn serving.

REMAINING SUSPECTS (need Rob/platform confirmation; NOT guessed as fact):
a. Docs custom-runtime example specifies BOTH healthcheck_endpoint AND readycheck_endpoint; our toml sets healthcheck_endpoint=/healthz but NO readycheck_endpoint. Unconfirmed whether a missing readycheck affects WS upgrade routing vs plain GET. WORTH VERIFYING against TOML reference.
b. Docs entrypoint example is a string ("uvicorn main:app --host 0.0.0.0 --port 5000"); ours is a list form (["python3","-u","main_stt.py"]). Both should run uvicorn, but whether Cerebrium's WS proxy keys off the documented uvicorn entrypoint form is unconfirmed.
c. Whether the DEPLOYED btg-stt revision actually applied [cerebrium.runtime.custom] (vs an earlier default ASGI runtime) — deploy/platform fact, check dashboard Settings/Code or redeploy logs.
d. disable_auth=true + ?bridge_auth= query — unlikely to cause 404 (would be 401/403); low priority.

NEXT (still read-only / Rob; NOTHING changed): 
- (AUTO-RUN) read Cerebrium TOML reference for runtime.custom required fields (esp. readycheck_endpoint) + any WS-specific routing note. 
- Likely fix candidate (to PROPOSE, not apply): add readycheck_endpoint and/or align entrypoint to the documented uvicorn string form in ops/cerebrium/stt/cerebrium.toml, then redeploy (confirm-before-live) and re-test /stream. This is a small toml change = AUTO-RUN draft + show-diff; deploy = confirm-before-live. Verify against TOML reference FIRST.

NOT done: no code/toml change, no deploy, no retry, no secret/env change, no upload/reprocess, no TTS/face.

## 2026-06-04T01:44:50Z — TOML reference check: readycheck/entrypoint suspects DOWNGRADED (both valid per docs). 404 cause is platform/deploy-layer WS upgrade not proxied — likely deployed revision didn't apply custom runtime, or needs Cerebrium support.
Fetched Cerebrium TOML reference (read-only). It eliminates the two toml-config suspects I floated and isolates the cause to the platform/deploy layer.

SOURCE: https://cerebrium.ai/docs/toml-reference/toml-reference

FINDINGS vs our ops/cerebrium/stt/cerebrium.toml:
- readycheck_endpoint: default "" => empty uses TCP readiness. Omitting it is VALID (TCP fallback), NOT required. A missing readycheck would not turn a WS upgrade into a plain GET 404. => suspect (a) DOWNGRADED, not the cause.
- entrypoint: type string[]; our ["python3","-u","main_stt.py"] is a valid shape (reference example is also a list). main_stt.py __main__ runs uvicorn.run(host=0.0.0.0, port=8003) — functionally equivalent to the docs' direct-uvicorn entrypoint; /healthz 200 proves uvicorn is serving. => suspect (b) DOWNGRADED.
- URL shape: reference confirms "all endpoints at .../v4/{project-id}/{app-name}/your/endpoint" => our wss://.../v4/p-a907d7c5/btg-stt/stream is CORRECT.

NET: WS is supported (custom runtime present + valid), URL correct, uvicorn bind correct, entrypoint + healthcheck valid, readycheck optional. Yet the dashboard shows /stream arriving as a plain GET -> 404. The cause is NOT in the visible repo config. Most plausible remaining explanations (need Rob/platform, NOT guessed as fact):
1. The DEPLOYED btg-stt revision did not actually apply [cerebrium.runtime.custom] (e.g. built before the block was added, or the deploy used a default ASGI runtime that doesn't proxy WS upgrade). Check dashboard Settings/Code/Builds for the active revision's runtime, or a clean redeploy.
2. A Cerebrium-side WS proxying nuance not documented (e.g. needs explicit websocket support flag, a different invocation, or readycheck must be HTTP 200 for WS routing). Would need Cerebrium support/Discord confirmation.

This is the limit of read-only docs research. Answer to "classify next path": CONFIG/RUNTIME-or-PLATFORM issue on the deployed app (NOT URL-only, NOT code-transport-change yet, NOT confirmed platform-incompatible — WS is supported).

PROPOSED next action (to PROPOSE only; nothing applied):
- Lowest-risk first: redeploy btg-stt unchanged (confirm-before-live) to ensure the current custom-runtime toml is the ACTIVE revision, then re-test /stream via dashboard Runs. If still 404, escalate to Cerebrium support with the Runs evidence (/healthz 200, /stream GET 404).
- Optional toml alignment to mirror docs exactly (low confidence it matters): set entrypoint to the uvicorn string-array form ["uvicorn","main_stt:app","--host","0.0.0.0","--port","8003"] and add readycheck_endpoint="/healthz". Small toml change = AUTO-RUN draft + show-diff; deploy = confirm-before-live. NOT applied.

NOT done: no code/toml change, no deploy, no retry, no secret/env change, no upload/reprocess, no TTS/face.

## 2026-06-04T01:53:36Z — Unchanged redeploy did NOT fix /stream 404. STOP per gate -> Cerebrium support escalation. Key: edge returns WS 101 but Cerebrium routing logs /stream as GET 404 — upgrade not reaching container.
Rob approved + executed option 1 (unchanged redeploy, confirm-before-live). Result: /healthz 200, /stream still 404. Did NOT fix it. STOP condition from the gate met -> prepare Cerebrium support escalation. No toml/code change, no retry, no second redeploy.

DEPLOY (unchanged, verified):
- toml clean since 46d523a (no code/toml change). Custom-runtime block intact: port=8003, entrypoint=["python3","-u","main_stt.py"], healthcheck_endpoint=/healthz.
- Redeploy build-0844d7dd LIVE; clean startup (whisper+VAD loaded, uvicorn on 0.0.0.0:8003, auth_secret_set=True).

SMOKE:
- /healthz -> HTTP 200 (33.6s cold start; scale-to-zero). model ready, cuda, int8_float16. PASS.
- /stream: WS upgrade probe (read-only handshake, no audio) -> socket-level HTTP/1.1 101 Switching Protocols (server: envoy). BUT dashboard Runs logged that exact request (18:52:02) as: /stream GET -> 404 (container 6fc5668ff7-kgdk9, 16ms). FAIL.

KEY INSIGHT (resolves the 101-vs-404 discrepancy):
- The 101 is the EDGE/Envoy completing a WS handshake at the gateway edge. Cerebrium's request-routing layer still records & routes /stream as a plain GET that 404s. The upgrade does NOT reach the container's @router.websocket("/stream"). So a socket-level 101 is NOT proof of end-to-end WS; the dashboard Runs 404 is the authoritative signal. This is why earlier direct 101 probes coexisted with backend 404s.
- Unchanged redeploy proves it is NOT a stale-revision / custom-runtime-not-applied issue: fresh revision, same 404. 

CONCLUSION: WS is documented-supported on v4 + our config/URL/bind valid, yet Cerebrium routes /stream as GET->404 to the container. This is a Cerebrium platform routing behavior we cannot resolve from the repo. ESCALATE to Cerebrium support.

ESCALATION EVIDENCE PACK (for Rob to send Cerebrium support/Discord):
- App: btg-stt (project p-a907d7c5, us-east-1, ADA_L4, custom runtime).
- toml runtime: [cerebrium.runtime.custom] port=8003, entrypoint=["python3","-u","main_stt.py"], healthcheck_endpoint="/healthz". FastAPI app (main_stt.py) includes @router.websocket("/stream") (no prefix) + @router.get("/healthz").
- Symptom: GET https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt/healthz -> 200. WS wss://.../v4/p-a907d7c5/btg-stt/stream -> dashboard Runs shows "/stream GET 404" though a raw TLS WS handshake returns 101 at the edge (server: envoy).
- Dashboard Runs rows: /stream GET 404 at 18:52:02 (build-0844d7dd, container 6fc5668ff7-kgdk9); /healthz GET 200 at 18:51:09 same container; same pattern pre-redeploy (18:15:30, 15:53:25).
- Question for Cerebrium: why is a wss:// request to a custom-runtime app's @app.websocket route routed/logged as a plain GET and returned 404 to the container? Is there a required flag/config for WS upgrade proxying on v4 sync endpoints, or a different endpoint host/path for WS?

NOT done: no toml/code change, no retry of 294c44ea, no second redeploy, no upload/reprocess, no TTS/face. STT blocked pending Cerebrium support answer.

## 2026-06-04T01:59:49Z — Google '/ws fixed-path' claim is WRONG (disproven by Cerebrium source). But it surfaces 2 real leads: @router-vs-@app WS mounting, and entrypoint form. Both = code/toml change (out of current scope).
Evaluated the Google answer against Cerebrium source docs (read-only). Its headline claim is incorrect; do NOT act on it as written. It does inadvertently point at two real structural deviations from the working example. No code/toml change made (out of scope this turn).

GOOGLE CLAIM (WRONG): WS must route to a fixed "/ws" ingress path (wss://.../<app>/ws); Cerebrium "proxies the /ws path specifically."
DISPROVEN BY SOURCE:
- Cerebrium WS docs (https://cerebrium.ai/docs/endpoints/websockets): URL last segment = your FastAPI @app.websocket("<path>") route name. Route name is arbitrary; the example uses /ws only because that app named its route /ws.
- Twilio example (https://cerebrium.ai/docs/v4/examples/twilio-voice-agent): defines @app.websocket("/ws") and connects to .../4-twilio-agent/ws. The /ws is that app's chosen route, NOT a magic ingress path.
- One of Google's citations was Oracle generative-AI docs (different platform) — answer stitched unrelated sources.
=> Renaming our route /stream -> /ws is NOT a justified fix. Not doing it.

TWO REAL LEADS (source-grounded, UNCONFIRMED — both are code/toml changes, gated):
1. WS route mounting: working Twilio example mounts the ws route DIRECTLY on the app: @app.websocket("/ws"). Our bridge uses @router.websocket("/stream") on an APIRouter + app.include_router(router) (main_stt.py:77, stt.py:205). Equivalent under normal ASGI, but it's a real deviation; whether Cerebrium ingress WS negotiation is sensitive to router-vs-app mounting is UNVERIFIED. This is the starkest difference from every working example -> highest-probability code fix candidate IF we try one before support.
2. Entrypoint form: every Cerebrium WS example uses entrypoint=["uvicorn","main:app","--host","0.0.0.0","--port",N] (uvicorn invoked directly). Ours is ["python3","-u","main_stt.py"] (uvicorn inside __main__). Both bind 0.0.0.0:8003 (/healthz 200 proves serving), but it's a deviation from the documented-working pattern.

CONFIDENCE: two plausible hypotheses, NEITHER confirmed. Not asserting a fix. Definitive answer still best from Cerebrium support (can confirm whether @router+include_router breaks their WS ingress). If Rob wants to try the highest-probability fix first, it's lead #1 (mount ws route on app), as a draft+show-diff, deploy confirm-before-live.

DECISION NEEDED (Rob): (A) escalate to Cerebrium support with the evidence pack (RESULTS 01:53Z) + these two leads, OR (B) let executor DRAFT lead #1 (and/or #2) as a show-diff for approval, then a confirm-before-live redeploy to test. Both are valid; A is lower-risk/definitive, B is faster if the hypothesis holds.

NOT done: no code change, no toml change, no route rename, no deploy, no retry, no upload/reprocess, no TTS/face. Per scope: stopped before any code/toml/deploy.

## 2026-06-04T02:03:10Z — BREAKTHROUGH (verified): backend WS route /api/call/ws/{id} WORKS through gateway (logged 1006 WS-close); btg-stt bare /stream does NOT (404). Differentiator = nested path vs bare single-segment. Both use @router+include_router, so router-mount eliminated.
Found a WORKING WebSocket precedent in our OWN backend, proving WS works on our v4 apps and isolating the differentiator. This eliminates the two prior leads and gives an evidence-backed (not certain) fix.

DECISIVE TEST (read-only WS probes + backend dashboard Runs):
- Probed backend's own ws route. Dashboard Runs logged:
  - /api/call/ws/test-probe (REAL route, prefix /api/call) -> STATUS 1006 (WebSocket close code) — upgrade REACHED the container; @router.websocket("/ws/{call_id}") ran and closed (no-auth -> websocket.close(4001), surfaced as 1006). 
  - /call/ws/test-probe and /ws/test-probe (non-routes) -> 403.
- Contrast btg-stt: /stream -> "/stream GET 404" (upgrade NOT reaching container).
- (All returned edge-101 at the socket; the dashboard status is authoritative. Edge-101 even on non-routes => socket-101 is meaningless for routing, as established.)

=> WebSockets DO work on our Cerebrium v4 apps. The backend's WS upgrade reaches its container; btg-stt's does not.

ELIMINATED (verified, not hypothesis):
- Router-vs-app mounting (prior lead #1): BACKEND also uses @router.websocket + app.include_router (call.py:417, main.py:188) and its WS WORKS. So @router+include_router is NOT the cause.
- "/ws magic path" (Google claim): already disproven; backend route is /ws/{call_id} under /api/call prefix, arbitrary.

VERIFIED DIFFERENTIATOR (source):
- Backend working WS route: call.router mounted prefix="/api/call" (main.py:188) -> path /api/call/ws/{call_id} (multi-segment, nested).
- btg-stt broken WS route: include_router(stt_svc.router) NO prefix (main_stt.py:77) -> bare single-segment /stream. (btg-stt /healthz is also bare + works, but it's GET, not WS.)
- Difference: working WS = nested multi-segment path; broken WS = bare single-segment path.

HONEST CAVEAT (NOT proven causal): path-depth is the cleanest difference, but other uncontrolled differences exist between the two apps — backend is CPU + min_replicas=1 (always-on); btg-stt is GPU(ADA_L4) + min_replicas=0 (scale-to-zero). Cannot fully exclude that the gateway's WS upgrade handling differs for scale-to-zero/GPU apps vs always-on CPU apps. Path-depth is the lowest-risk first fix; if it fails, the next suspect is scale-to-zero.

PROPOSED FIX (to DRAFT as show-diff; NOT applied; needs Rob approval + confirm-before-live redeploy):
- Mirror the backend's working pattern: give the STT bridge router a prefix so the WS route is multi-segment, e.g. include_router(stt_svc.router, prefix="/api/stt") -> route /api/stt/stream (or /v1/stream). 
- Update backend client _ws_url_from (stt_bridge.py:56-72) to build the new path.
- Two small reversible edits; no STT logic change. healthz can stay or move with it.
- Then confirm-before-live redeploy btg-stt + backend; re-test /stream-equivalent via dashboard Runs (expect a WS-close status, not GET 404), then a real reprocess.

ALTERNATIVE if path-depth fails: set btg-stt min_replicas=1 (always-on, ROB-ONLY spend ~$576/mo — exceeds $100 ceiling, so only as a SHORT test then revert) to test the scale-to-zero hypothesis; or escalate.

NOT done: no code/toml change, no deploy, no retry, no upload/reprocess, no TTS/face. Decision + draft-approval needed.

## 2026-06-04T02:08:15Z — Applied + committed /api/stt prefix fix (3 files, backtogether d30301e). Not deployed. Awaiting confirm-before-live for btg-stt then backend.
Rob approved "apply it". Applied the 3 drafted edits exactly, verified, committed to backtogether repo. No deploy yet.

APPLIED + VERIFIED (greps + full diff reviewed):
- bridge/main_stt.py:82 (was 77): app.include_router(stt_svc.router, prefix="/api/stt") + explanatory comment.
- ops/cerebrium/stt/main_stt.py:82 (deployed snapshot): SAME edit (identical diff).
- backend/app/services/clients/stt_bridge.py _ws_url_from: all 3 return branches now build /api/stt/stream; docstring updated.
- git status: ONLY these 3 files changed (no unrelated edits). 18 insertions / 6 deletions.
- Net: ws route /stream -> /api/stt/stream. Bare @app.get("/healthz") (Cerebrium healthcheck) unchanged.

COMMIT: backtogether repo, branch main, hash d30301e. tree clean. NOT pushed (push not in approved scope; commit only as instructed).

SCOPE-CHECK NOTE (flagged, NOT acted on): TTS + avatar bridges already build NESTED ws paths — tts_bridge.py:91-94 -> /tts/stream; avatar_bridge.py:86-89 -> /avatar/stream. They are multi-segment already (though only one level deep, /tts/stream not /api/tts/stream). Whether one-level-deep is "nested enough" for the gateway is UNVERIFIED until the /api/stt/stream test proves the path-depth theory. Did not touch TTS/avatar (out of scope).

NEXT (ROB-ONLY, confirm-before-live, SEPARATELY per the approval):
1. Redeploy btg-stt (picks up snapshot prefix change). Smoke: /healthz 200; WS probe to /api/stt/stream -> dashboard Runs shows WS-close status (NOT GET 404).
2. Redeploy backend (picks up client _ws_url_from change). Smoke: /api/health 200.
   - Order matters: deploy btg-stt FIRST (new route exists), THEN backend (client points at new route). If backend deploys first it would call /api/stt/stream on an old btg-stt that still only has /stream -> 404; deploying stt first avoids a transient mismatch.
3. After both live: real reprocess of a fresh video -> audio_extracted -> transcribed, source_speaker_segments>0, dashboard shows /api/stt/stream WS activity.
4. If /api/stt/stream STILL 404: path-depth disproven -> test btg-stt min_replicas=1 (SHORT ROB-ONLY spend, then revert) or escalate.

NOT done: no deploy, no push, no upload/reprocess/retry, no TTS/face. Stopped for confirm-before-live deploy approval (btg-stt first, then backend).

## 2026-06-04T02:14:43Z — Prefix fix FALSIFIED: /api/stt/stream STILL logs GET 404 after btg-stt redeploy. Path-depth was NOT the cause. HALT — did not deploy backend. Scale-to-zero now the leading suspect.
Deployed btg-stt with the /api/stt prefix fix. Dashboard Runs: /api/stt/stream STILL logs as "GET 404". Path-depth hypothesis is FALSIFIED by a real deploy test. HALT per gate — did NOT deploy backend, did NOT reprocess.

DEPLOY + SMOKE (btg-stt build-da6a8e30, live):
- /healthz -> HTTP 200 (32.97s cold start). model ready cuda int8_float16.
- WS probe to /api/stt/stream -> edge 101 (not authoritative).
- DASHBOARD RUNS (authoritative): /api/stt/stream GET 404 at 19:13:18 (container 67794f77bd-xkc4b, 26ms). Same failure mode as bare /stream. /healthz 200 at 19:12:45 same container.
=> The prefix change did NOT fix it. WS upgrade to /api/stt/stream still routed as plain GET -> 404.

CONCLUSION: path-depth (nested vs bare path) was NOT the cause. My hypothesis was wrong; the backend-vs-stt contrast had a confound I could not exclude and it turned out to be the confound, not the path.

REMAINING DIFFERENTIATOR (the confound, now leading suspect): backend (WS WORKS) is CPU + min_replicas=1 (ALWAYS-ON); btg-stt (WS 404) is GPU ADA_L4 + min_replicas=0 (SCALE-TO-ZERO). Note: the 404 probe hit an already-warm container (same container as the prior /healthz 200), and failed fast (26ms) — so it is not a simple "cold start drops the upgrade." It may be that scale-to-zero apps get a different gateway routing profile (e.g. sync-request routing that doesn't proxy WS upgrades) vs always-on apps. UNVERIFIED.

CODE STATE: prefix fix committed (backtogether d30301e) but INEFFECTIVE. Options: (a) leave it (harmless, and may still be needed in combination), or (b) revert. Recommend LEAVE for now — it is not wrong, just insufficient; reverting adds churn. Backend NOT deployed, so backend client still builds /stream — consistent with stt still effectively broken either way. (If we later confirm the real fix, reconcile the path then.)

OPTIONS (Rob decision):
A. Test scale-to-zero hypothesis: set btg-stt min_replicas=1 (always-on, like the working backend) + redeploy, re-probe /api/stt/stream via Runs. This is the cleanest test of the leading suspect. COST: always-on ADA_L4 ~$0.80/hr (~$576/mo) — exceeds the $100 ceiling, so ONLY as a SHORT test then revert to 0. ROB-ONLY (spend).
B. Escalate to Cerebrium support with the now-stronger evidence: WS works on our always-on CPU app (/api/call/ws -> 1006) but 404s on our scale-to-zero GPU app (/api/stt/stream and /stream both GET 404), same project, same @router+include_router pattern. Definitive but slow.
C. Both in parallel: A as the fast test while B runs.

NOT done: backend NOT deployed, no reprocess, no min_replicas change, no push, no TTS/face. Stopped at the gate HALT.

## 2026-06-04T02:28:09Z — Scale-to-zero hypothesis ALSO FALSIFIED: /api/stt/stream STILL GET 404 with min_replicas=1 (always-on). Reverted to min_replicas=0 (deployed). Both hypotheses dead -> Cerebrium escalation (plan C / leg B).
Short bounded scale-to-zero test (Rob-approved). Set btg-stt min_replicas=1, redeployed, probed /api/stt/stream. STILL GET 404. Scale-to-zero is NOT the cause. Reverted to min_replicas=0 and redeployed (cost-stop honored). Escalate to Cerebrium.

TEST (min_replicas=1, always-on, build-26a9af59):
- /api/stt/stream -> dashboard Runs GET 404 at 19:20:42 (container 7c94ccf469-fwmhs, always-on replica). Same failure mode.
=> scale-to-zero vs always-on is NOT the differentiator. Both my hypotheses (path-depth AND scale-to-zero) now FALSIFIED by real deploy tests.

COST-STOP (honored):
- Reverted toml min_replicas 1 -> 0; redeployed build-ce08b0a7 (live). toml back to committed baseline (git clean). Always-on window ~6 min (19:16->19:23), a few cents. btg-stt is scale-to-zero again. (Dashboard visual re-confirm was denied/declined; the binding control is the toml+deploy, which is done.)

STATE OF HYPOTHESES — both dead:
- Path-depth: FALSIFIED (/api/stt/stream 404 same as bare /stream).
- Scale-to-zero: FALSIFIED (/api/stt/stream 404 even at min_replicas=1).
- Router-vs-app mounting: already eliminated (backend uses @router+include_router and WS works).
- Remaining unexplained difference between working backend WS and broken btg-stt WS: CPU (backend) vs GPU/ADA_L4 (btg-stt), and/or disable_auth=true on btg-stt (backend likely false), and/or some per-app gateway config. NOT TESTED — do not guess further; escalate.

ESCALATION TO CEREBRIUM (plan C leg B — now the path) — evidence pack:
- Same project p-a907d7c5, same FastAPI @router.websocket + app.include_router pattern.
- WORKS: backend (CPU, min_replicas=1, disable_auth likely false) — /api/call/ws/{id} -> WS reaches container (Runs status 1006).
- FAILS: btg-stt (GPU ADA_L4, tested at BOTH min_replicas=0 and =1, disable_auth=true) — /stream AND /api/stt/stream both -> "GET 404", upgrade routed as plain GET, never reaches container. /healthz GET 200 fine.
- Question for Cerebrium: why does a wss:// upgrade to a custom-runtime app's @app.websocket route get routed/logged as a plain GET and 404'd to the container on btg-stt, while the same pattern works on our backend app? What app-level config enables WS upgrade proxying (e.g. does disable_auth, GPU runtime, or a specific runtime flag affect WS ingress)?
- NEW SUSPECT worth Rob checking/asking: disable_auth=true on btg-stt vs the backend. If Cerebrium's WS upgrade proxying requires the default auth path (or a specific flag) that disable_auth bypasses, that would explain WS-on-backend vs WS-404-on-stt. UNVERIFIED — candidate question for Cerebrium AND a possible next cheap test (toggle disable_auth) if Rob wants.

CODE STATE: /api/stt prefix fix still committed (d30301e), still ineffective alone; LEFT in place (harmless). toml reverted to min_replicas=0 (uncommitted net-zero; matches committed baseline).

NOT done: backend NOT deployed, no reprocess/upload/retry, no TTS/face, no push. Always-on reverted. STT blocked pending Cerebrium answer (or a disable_auth test if Rob approves).

## 2026-06-04T02:33:57Z — GPU-app WS diagnostic: qwen has NO ws route (vLLM HTTP-only) so GPU-vs-CPU stays n=1. But btg-stt bogus-path test PROVES app-wide WS-upgrade non-proxying: real route + bogus path BOTH log GET 404 (backend distinguished them: 1006 vs 403).
Ran the GPU-app WS diagnostic (read-only). Could not strengthen GPU-vs-CPU (no second GPU app has a ws route), but a bogus-path comparison on btg-stt proved the failure is app-wide WS-upgrade non-proxying, route-independent.

APP INVENTORY (verified, only 3 deployed apps):
- backtogether-backend: CPU, ws route /api/call/ws/{id} -> WS WORKS (1006).
- btg-llm-qwen-4b: GPU AMPERE_A10, vLLM HTTP/OpenAI only, NO ws route -> cannot test WS here.
- btg-stt: GPU ADA_L4, ws route /api/stt/stream -> 404.
=> No second GPU-app-with-ws exists; GPU-vs-CPU remains a correlation (n=1 each side), NOT a proven mechanism. Did not overstate.

BOGUS-PATH TEST (the informative part, btg-stt, read-only):
- WS probe to /api/stt/does-not-exist-zzz (no such route) -> dashboard Runs: GET 404 (19:31:47, 35.9s cold start).
- WS probe to /api/stt/stream (real route) -> GET 404 (19:20:42).
- BOTH identical: GET 404. 
- CONTRAST backend: bogus path -> 403; real ws route -> 1006 (WS-close). Backend gateway DISTINGUISHES ws-upgrade from non-route.
- CONCLUSION (proven): btg-stt's gateway does NOT negotiate the WS upgrade for ANY path — it forwards everything as plain GET, so no @websocket route can ever match -> 404. Failure is APP-WIDE WS-upgrade non-proxying on btg-stt, independent of route name, path depth, scale-to-zero/always-on (all tested).

WHAT'S PROVEN vs NOT:
- PROVEN: btg-stt does not proxy WS upgrades at all; backend (CPU) does.
- NOT PROVEN: that GPU runtime per se is the cause (no GPU+ws control app). Could be GPU runtime, or a per-app gateway/runtime config difference between how btg-stt vs backend were created/deployed.

COST: bogus probe cold-started btg-stt (now briefly running); min_replicas=0 so it scales back to 0 on cooldown. No lingering spend.

ESCALATION (sharpened for Cerebrium): On project p-a907d7c5, app btg-stt (GPU ADA_L4, custom runtime, disable_auth=true), wss:// upgrades to ANY path (real @app.websocket route or nonexistent) are routed/logged as plain GET -> 404; /healthz GET works (200). The identical FastAPI @router.websocket + include_router pattern on our CPU app backtogether-backend negotiates WS correctly (real ws route logs 1006, nonexistent logs 403). What enables WS-upgrade proxying on a GPU custom-runtime app? Is it unsupported, or gated by a flag/config our backend has and btg-stt lacks?

NOT done: backend NOT deployed, no reprocess/upload, no TTS/face, no push, no further spend test. min_replicas back at 0. STT blocked pending Cerebrium answer or an architectural transport change.

## 2026-06-04T02:37:58Z — Drafted option B (HTTP transcribe for upload path). Verified all assumptions against source: header auth (X-Bridge-Auth), httpx house pattern, raw-body (no multipart dep), faster-whisper decodes path. NOT applied.
Drafted the HTTP-transcribe fix for the upload/batch STT path (sidesteps the WS gateway 404). Every assumption verified read-only against source before finalizing. Nothing applied; awaiting Rob approval.

WHY THIS WORKS: the blocked path is upload reprocess = batch transcription of a finite extracted .wav. It does not need WS streaming. A POST /api/stt/transcribe routes as normal HTTP (which the Cerebrium gateway proxies fine — /healthz GET 200, backend HTTP endpoints work), avoiding the WS-upgrade-not-proxied issue entirely.

DRAFT (2 files + snapshot = 3 edit sites):
1. bridge/services/stt.py (+ ops/cerebrium/stt/services/stt.py snapshot):
   - import: add Request, Depends; add `from auth import require_bridge_auth` (HTTP variant; stt.py already does `from auth import require_bridge_auth_ws` at line 64, so path works; auth.py is flat in the snapshot dir — confirmed).
   - add _transcribe_file_blocking(path, language): _whisper.transcribe(path, vad_filter=True, beam_size=1) -> (text, duration). faster-whisper decodes the file path itself (no client ffmpeg).
   - add @router.post("/transcribe", dependencies=[Depends(require_bridge_auth)]) reading raw request.body() (NOT multipart -> avoids missing python-multipart dep), writes temp .wav, transcribes on a thread, returns {text, duration_s}. Lands at /api/stt/transcribe (prefix already committed d30301e).
2. backend/app/services/clients/stt_bridge.py transcribe_file (line 248): rewrite to httpx POST base+"/api/stt/transcribe" with params={language}, content=<file bytes>, headers X-Bridge-Auth (mirrors tts_bridge/face_bridge _auth_headers + httpx.AsyncClient house pattern). timeout=1800 (35-min files). Returns obj["text"]. Raises on failure (processing.py captures).

VERIFIED ASSUMPTIONS (read-only):
- auth: HTTP bridge auth is HEADER-based require_bridge_auth (auth.py:44, X-Bridge-Auth Header), NOT ?bridge_auth= query (that's the WS variant). Client must send header. (Corrected from first draft.)
- client lib: backend uses httpx==0.27.2 (tts/face/avatar bridges all import httpx). NOT aiohttp.
- multipart: bridge toml has fastapi but NOT python-multipart -> UploadFile would crash. Raw request.body() avoids it. (Corrected.)
- faster-whisper: _whisper.transcribe accepts a file PATH and decodes internally; vad_filter=True for whole-file.
- WS path: KEEP /stream + open_stream + streaming machinery (realtime-call path uses it; separate concern). This change touches ONLY transcribe_file (upload path).
- route reachability: /transcribe is POST -> gateway proxies normal HTTP (proven: /healthz GET 200). This is the crux of why it sidesteps the WS bug.

OPEN/NOTE: 35-min file -> single _whisper.transcribe call on GPU; timeout 1800s client-side. faster-whisper handles long files (segments generator). Acceptable for batch. If too slow, chunk later (not now).

NEXT (ROB approval): "apply it" -> executor applies 3 edits, shows greps/diff, commits (backtogether), then confirm-before-live redeploy btg-stt (new route) THEN backend (new client). Then ROB reprocess a fresh video -> executor verifies transcribed + segments>0.

NOT done: nothing applied, no deploy, no reprocess, no TTS/face, no push. Draft only.
