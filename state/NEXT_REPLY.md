# NEXT REPLY

status: PENDING
updated_at: 2026-06-04T03:05:00Z
consumed_at:
consumed_by:
gate_commit: 099c822
classification_hint: ROB-ONLY reprocess trigger with HTTP status capture

## body
The HTTP STT path is deployed, but the previous reprocess trigger did not reach or complete the handler. Do not interpret that as an STT pass/fail. The row did not reset, no new job_ledger row appeared, and no /api/stt/transcribe call appeared. The missing item is the HTTP status and response body from Rob's authed POST.

Rob-only next action: rerun the reprocess POST and capture HTTP status plus response body. Do not paste tokens.

Use this exact local command with TOKEN already set locally:

API='https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend'
SOURCE_ID='294c44ea-9784-42eb-988a-701a11d7c448'

curl -sS -X POST "$API/api/upload/_reprocess/$SOURCE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -w '\nHTTP_STATUS=%{http_code}\n'

Expected success shape:
{"status":"reset_and_enqueued","enqueued":true,"job_id":"..."}
HTTP_STATUS=200

If the response is non-2xx, report the status and body exactly, but never paste the token. Common causes would be 401 auth/session token issue, 404 wrong endpoint/path/source id, or 5xx backend error.

After Rob provides a 2xx reset_and_enqueued response, executor verifies read-only:
- source row updated/reset and advances beyond audio_extracted
- new job_ledger row exists for source 294c44ea
- btg-stt Runs shows POST /api/stt/transcribe
- transcript_text populated and source_speaker_segments > 0, or capture exact error if it fails
- update RESULTS.md and CURRENT_GATE.md

Hard constraints:
- executor does not call reprocess
- do not upload/reprocess again without Rob
- do not deploy
- do not start TTS or face/avatar
- do not read, print, or set secrets
