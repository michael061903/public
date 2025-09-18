#!/usr/bin/env bash
set -euo pipefail

## ------------------- USER SETTINGS -------------------
SERVER_HOST="171.206.x.x"
DEST_DIR="/storage/PUT_STUFF_IN_HERE/NetOpsLB/QKVIEWS"
DEVICES_FILE="./devices.txt"          # one F5 IP/host per line
LOCAL_STAGING="/tmp/qkviews"          # temp download directory
SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=20"
## -----------------------------------------------------

require() { command -v "$1" >/dev/null 2>&1 || { echo "ERROR: '$1' is required." >&2; exit 1; }; }
require ssh
require scp
require sshpass

mkdir -p "$LOCAL_STAGING"

# --- safepass-aware secret prompts ---
# If 'safepass' exists, use it for prompts; otherwise fall back to read -s
prompt_secret() {
  local label="$1" varname="$2"
  if command -v safepass >/dev/null 2>&1; then
    # Expect safepass to print the secret to stdout after an interactive prompt.
    # Example: safepass "Enter F5 password"
    local value
    value="$(safepass "$label")"
    if [[ -z "${value}" ]]; then
      echo "ERROR: No value returned from safepass for '$label'." >&2
      exit 1
    fi
    printf -v "$varname" '%s' "$value"
  else
    read -r -p "$label username: " user
    read -r -s -p "$label password: " pass; echo
    printf -v "${varname}_USER" '%s' "$user"
    printf -v "${varname}_PASS" '%s' "$pass"
  fi
}

# We need two credential sets:
# 1) F5 elevated credentials
# 2) Scripting server standard credentials
if command -v safepass >/dev/null 2>&1; then
  # Ask for each item explicitly (user + pass) via safepass to keep symmetry
  F5_USER="$(safepass 'F5 username')"
  F5_PASS="$(safepass 'F5 password')"
  SRV_USER="$(safepass 'Scripting server username')"
  SRV_PASS="$(safepass 'Scripting server password')"
  [[ -n "$F5_USER" && -n "$F5_PASS" && -n "$SRV_USER" && -n "$SRV_PASS" ]] || { echo "ERROR: Missing creds from safepass."; exit 1; }
else
  echo "NOTE: 'safepass' not found; falling back to secure prompts."
  read -r -p "F5 username: " F5_USER
  read -r -s -p "F5 password: " F5_PASS; echo
  read -r -p "Scripting server username: " SRV_USER
  read -r -s -p "Scripting server password: " SRV_PASS; echo
fi

# Verify destination directory exists (create if we are running on the server).
# If we’re NOT on the server, we’ll create it remotely after we connect.
if [[ "$(hostname -I 2>/dev/null || true)" == *"$SERVER_HOST"* ]]; then
  mkdir -p "$DEST_DIR"
else
  echo "Ensuring destination directory exists on ${SERVER_HOST}:${DEST_DIR} ..."
  sshpass -p "$SRV_PASS" ssh $SSH_OPTS "${SRV_USER}@${SERVER_HOST}" "mkdir -p '$DEST_DIR'"
fi

timestamp() { date '+%Y-%m-%d %H:%M:%S'; }

echo "==== $(timestamp) Starting qkview collection ===="
echo "Devices file: $DEVICES_FILE"
echo

# Read device list and process each F5
while IFS= read -r DEVICE || [[ -n "${DEVICE}" ]]; do
  # Skip empties/comments
  [[ -z "$DEVICE" || "$DEVICE" =~ ^# ]] && continue

  echo "---- $(timestamp) Processing $DEVICE ----"

  # Build a remote filename using the F5's hostname and current timestamp (on the F5)
  REMOTE_CMD='HN=$(hostname); TS=$(date +%Y%m%d-%H%M%S); OUT="/var/tmp/${HN}_${TS}.qkview"; \
              echo "Generating qkview: $OUT"; \
              qkview -s0 -f "$OUT" >/dev/null 2>&1 || qkview -s0 > /dev/null; \
              # If -f not supported/failed, find newest qkview
              if [[ ! -f "$OUT" ]]; then OUT=$(ls -1t /var/tmp/*.qkview 2>/dev/null | head -n1); fi; \
              echo "$OUT"'
  set +e
  REMOTE_OUT_PATH="$(sshpass -p "$F5_PASS" ssh $SSH_OPTS "${F5_USER}@${DEVICE}" "$REMOTE_CMD")"
  rc=$?
  set -e
  if [[ $rc -ne 0 || -z "$REMOTE_OUT_PATH" ]]; then
    echo "ERROR: Failed to generate qkview on ${DEVICE}."
    continue
  fi

  # Extract the last line (echo path)
  REMOTE_QKV_PATH="$(echo "$REMOTE_OUT_PATH" | tail -n1 | tr -d '\r')"
  BASENAME="$(basename "$REMOTE_QKV_PATH")"
  LOCAL_FILE="${LOCAL_STAGING}/${DEVICE//[:\/]/_}_${BASENAME}"

  echo "Remote qkview path on ${DEVICE}: ${REMOTE_QKV_PATH}"
  echo "Downloading to local staging: ${LOCAL_FILE}"

  # Pull to local staging
  sshpass -p "$F5_PASS" scp $SSH_OPTS "${F5_USER}@${DEVICE}:${REMOTE_QKV_PATH}" "$LOCAL_FILE"

  # Ship to scripting server destination
  echo "Uploading to ${SERVER_HOST}:${DEST_DIR}/"
  sshpass -p "$SRV_PASS" scp $SSH_OPTS "$LOCAL_FILE" "${SRV_USER}@${SERVER_HOST}:${DEST_DIR}/"

  # Optional: clean up remote (comment out if you want to keep qkviews on the F5)
  set +e
  sshpass -p "$F5_PASS" ssh $SSH_OPTS "${F5_USER}@${DEVICE}" "rm -f '${REMOTE_QKV_PATH}'" >/dev/null 2>&1
  set -e

  echo "OK: ${DEVICE} -> ${DEST_DIR}/${BASENAME}"
  echo
done < "$DEVICES_FILE"

echo "==== $(timestamp) Done. Files are on ${SERVER_HOST}:${DEST_DIR}/ ===="





-----------------------------------------------------------------------------------------------------------------------------------------


from flask import Flask, request, render_template_string
import subprocess

app = Flask(__name__)

HTML = '''
<h1>Load Balancer Command Builder</h1>
<form method="post">
    F5 Username: <input type="text" name="f5_user"><br>
    F5 Password: <input type="password" name="f5_pass"><br>
    F5 Device IP/Hostname: <input type="text" name="f5_host"><br>
    TMOS Command: <input type="text" name="tmos_cmd"><br>
    BASH Command: <input type="text" name="bash_cmd"><br>
    <input type="submit" value="Run Script">
</form>

{% if output %}
<br><br><br>
<h2>Script Output:</h2>
<pre>{{ output }}</pre>
{% endif %}
'''

@app.route('/', methods=['GET', 'POST'])
def index():
    output = None
    if request.method == 'POST':
        f5_user = request.form['f5_user']
        f5_pass = request.form['f5_pass']
        f5_host = request.form['f5_host']
        tmos_cmd = request.form['tmos_cmd']
        bash_cmd = request.form['bash_cmd']

        result = subprocess.run(
            ["bash", "run_single_device.sh", f5_user, f5_pass, f5_host, tmos_cmd, bash_cmd],
            capture_output=True,
            text=True
        )
        output = result.stdout
    return render_template_string(HTML, output=output)
