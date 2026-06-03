# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate the STT path end-to-end. BLOCKED upstream of STT: ellis video sources are not retrievable. Rob to choose how to get a readable source in front of STT.

## Done / verified
- STT bridge btg-stt READY (/healthz 200, cuda, auth_secret_set=True).
- Backend wired + redeployed rev 00023; /api/health 200; WS routing to btg-stt /stream confirmed (HTTP 101).
- Reprocess of all 5 ellis video sources: all FAILED upstream of STT (FileNotFoundError on source file). See RESULTS 23:12Z.

## NEW BLOCKER (this is why STT is still unvalidated)
- The 5 ellis video sources have storage_uri = file:///opt/backtogether/uploads/... (local disk, prior deployment), NOT r2://.
- Current backend container has no local copy; ostore can't fetch a file:// URI from R2; processing.py:156 raises FileNotFoundError before audio extraction / STT.
- STT bridge logs show no transcription requests (pipeline died upstream). STT wiring is NOT disproven, just not yet exercised.
- Retrying reprocess will fail identically — NOT a retry case.

## Next step — Rob chooses ONE (then executor verifies read-only)
- A. Re-upload the 5 ellis videos (fresh upload -> R2 with r2:// uri) -> reprocess -> verify. Clean path to exercise STT on the real persona.
- B. Backfill: check if original local files survive on any persistent volume and can be pushed to R2 (uncertain; container disks may be gone).
- C. Smallest validation: new test persona + one fresh small video upload -> reprocess -> verify STT end-to-end, independent of the orphaned ellis sources.
Recommendation (per standing rule 2, smallest validating step): C, then revisit ellis re-upload (A) for real persona data.
NOTE: uploads/re-uploads are ROB-ONLY (auth + prod data + object storage). Executor verifies read-only after.

## After STT validates
- TTS bridge bring-up (voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval.

## ROB-ONLY (carried)
- Source upload/re-upload (auth as Rob, prod data, R2 writes).
- Reprocess trigger (prod DB mutation + auth as Rob).
- TTS/face GPU deploy = new paid infra.
- No secret values set by executor.

## Hard constraints
No prod DB writes / uploads by executor. No calls placed/ended by executor. No backend redeploy without confirm-before-live. No blind retry. No TTS/face this gate.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
