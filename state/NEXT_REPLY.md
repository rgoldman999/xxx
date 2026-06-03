# NEXT REPLY

status: PENDING
updated_at: 2026-06-03T23:40:00Z
consumed_at:
consumed_by:
gate_commit: 8cda5e331eeb95278e37f641011c82a8056487c6
classification_hint: ROB-ONLY R2 secret setup, then confirm-before-live backend redeploy

## body
Current gate has changed: STT is healthy and backend routing to btg-stt is verified, but STT validation is blocked before transcription because existing Ellis uploads are stored as file:// local container paths. Those files disappeared across backend redeploys. R2 object storage must be enabled before any upload-based STT validation can succeed.

Rob action required first (ROB-ONLY): set backend R2 configuration values. Secret/config VALUES must stay local and must not be pasted into chat.

Required backend config names from config.py / gate:
- r2_enabled=true
- r2_endpoint
- r2_access_key
- r2_secret_key
- r2_bucket

Suggested local pattern for Rob only, replacing placeholders locally and not pasting values into chat:

# Set these locally only
R2_ENDPOINT="<your-r2-endpoint>"
R2_ACCESS_KEY="<your-r2-access-key>"
R2_SECRET_KEY="<your-r2-secret-key>"
R2_BUCKET="<your-r2-bucket>"

# Use the working Cerebrium CLI path if cerebrium is not on PATH
~/Library/Python/3.13/bin/cerebrium secrets add r2_enabled="true"
~/Library/Python/3.13/bin/cerebrium secrets add r2_endpoint="$R2_ENDPOINT"
~/Library/Python/3.13/bin/cerebrium secrets add r2_access_key="$R2_ACCESS_KEY"
~/Library/Python/3.13/bin/cerebrium secrets add r2_secret_key="$R2_SECRET_KEY"
~/Library/Python/3.13/bin/cerebrium secrets add r2_bucket="$R2_BUCKET"

# Verify names only, not values
~/Library/Python/3.13/bin/cerebrium secrets list | grep -E 'r2_enabled|r2_endpoint|r2_access_key|r2_secret_key|r2_bucket'

If any secret already exists, use the Cerebrium update/set equivalent or report the exact error; do not paste secret values.

After Rob confirms the five R2 names are set, executor should proceed from the current gate:
1. confirm-before-live backend redeploy to pick up R2 config, using the deploy spec already carried in the gate/policy
2. run backend health smoke
3. verify with a fresh small upload that storage_uri starts with r2:// rather than file://
4. update RESULTS.md and CURRENT_GATE.md with the outcome
5. stop at the next ROB-ONLY boundary before any larger re-upload/reprocess

Hard constraints:
- do not paste secret values
- executor does not set secret values
- no upload/re-upload by executor unless a later gate explicitly permits it
- no blind retry of the old file:// Ellis sources
- no TTS or face/avatar in this gate
- no backend deploy unless the gate includes target, command, smoke, rollback, and confirm-before-live
