# NEXT REPLY

status: PENDING
updated_at: 2026-06-04T01:25:00Z
consumed_at:
consumed_by:
gate_commit: 50ee34c0681c7efd123fa38bc257eb48e0b407ef
classification_hint: AUTO-RUN read-only diagnosis; likely ROB-ONLY backend secret/env fix then confirm-before-live deploy

## body
Rob ran the authenticated inspect call for source 294c44ea-9784-42eb-988a-701a11d7c448. Key result: R2 is working, audio extracted, but the backend reports `env_stt_bridge_url_present: false`.

Important inspect facts:
- storage_uri is r2://backtogether-prod/...
- audio_storage_uri is r2://backtogether-prod/...wav
- audio_duration_seconds is 2085.941438
- processing_status failed
- processing_stage audio_extracted
- error_message empty
- transcript_len 0
- env_hf_token_present false
- env_stt_bridge_url_present false

Interpretation:
The current blocker is very likely backend configuration for the STT bridge, not R2. The backend does not see the STT bridge URL under whatever env/config key `_inspect` checks. Do not continue trying R2, and do not blindly retry this source.

Next action for executor:
1. Do read-only code/config diagnosis only.
2. Inspect backend config and upload processing code to identify the exact expected env/config name for the STT bridge URL and the exact env/config name `_inspect` reports as env_stt_bridge_url_present.
3. Compare that expected name to whatever was previously set for STT_SERVICE_URL / stt bridge endpoint in the gate/docs. Do not read or print secret values.
4. Determine whether the backend needs a lower-case config key, different secret name, or redeploy to pick it up.
5. Update RESULTS.md and CURRENT_GATE.md with the exact expected secret/env name and Rob-only set command if a secret/env must be set.
6. Stop at the ROB-ONLY boundary if a secret/env value needs to be set, or before any backend redeploy unless the gate carries confirm-before-live deploy fields.

Likely thing to verify:
- The inspect output says env_stt_bridge_url_present=false. The backend may be expecting a config field like stt_bridge_url / STT_BRIDGE_URL, not STT_SERVICE_URL. Confirm from source before changing anything.

Known STT bridge endpoint shape, from prior successful STT smoke:
https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/btg-stt

Hard constraints:
- do not retry/reprocess source 294c44ea until the missing STT bridge URL config is understood/fixed
- do not upload/reprocess
- do not start TTS or face/avatar
- do not read, print, or set secret values
- no backend redeploy without confirm-before-live
- no blind retries
