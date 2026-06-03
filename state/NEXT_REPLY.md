# NEXT REPLY

status: PENDING
updated_at: 2026-06-03T23:20:00Z
consumed_at:
consumed_by:
gate_commit: 81500f61d99662c98c16742a89ecc7749f08cbe0
classification_hint: ROB-ONLY trigger, then AUTO-RUN verification

## body
Rob needs to trigger the Ellis STT reprocess as the authenticated owner, because this endpoint mutates prod DB and requires Rob auth. Executor must not call it.

After Rob confirms the reprocess was triggered, executor should verify read-only:
- persona_sources video rows advance from audio_extracted to transcribed or processing_status complete
- source_speaker_segments for Ellis becomes greater than zero
- btg-stt bridge logs show transcription websocket requests landing
- if any row errors again, capture exact error and stop; do not blind retry
- update RESULTS.md and CURRENT_GATE.md with the outcome

Reprocess endpoint base:
https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend/api/upload/_reprocess/

Video source IDs:
- 1154ccfa-1bb4-4a2a-8e27-62bc3665eb95
- 368930f7-fdf0-411b-93ee-76a6141c78b9
- 7fa2e508-aea4-4ecf-ad1d-abe19eddd3e6
- d9caf45c-81bb-4630-a92d-febf527938a9
- f149bd58-a66f-462f-a51c-10fe50614783

Suggested local command for Rob only, with TOKEN set locally and not pasted into chat:

API="https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend"
for SOURCE_ID in \
  1154ccfa-1bb4-4a2a-8e27-62bc3665eb95 \
  368930f7-fdf0-411b-93ee-76a6141c78b9 \
  7fa2e508-aea4-4ecf-ad1d-abe19eddd3e6 \
  d9caf45c-81bb-4630-a92d-febf527938a9 \
  f149bd58-a66f-462f-a51c-10fe50614783
do
  echo "Reprocessing $SOURCE_ID"
  curl -sS -X POST "$API/api/upload/_reprocess/$SOURCE_ID" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json"
  echo
done

Hard constraints:
- do not start TTS
- do not start face/avatar
- executor performs read-only verification only after Rob triggers reprocess
- do not blind retry failed rows
