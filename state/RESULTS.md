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
