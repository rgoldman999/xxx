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
