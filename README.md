```bash
from flask import Flask, request, render_template_string, redirect, url_for
import subprocess
import os

app = Flask(__name__)

HTML = """
<!doctype html>
<html>
  <head>
    <title>qkview Runner</title>
    <meta charset="utf-8" />
    <style>
      body { font-family: Arial, sans-serif; margin: 30px; }
      textarea { width: 600px; height: 200px; }
      input[type=text], input[type=password] { width: 400px; }
      pre { background: #f4f4f4; padding: 15px; border-radius: 6px; max-width: 90%; overflow: auto; }
      .note { color: #666; font-size: 0.9em; }
    </style>
  </head>
  <body>
    <h1>qkview Runner - Web Front</h1>

    <p class="note">Enter one device (FQDN or IP) per line in the device list.</p>

    <form method="post">
      <label>Elevated username:<br>
        <input type="text" name="username" required>
      </label>
      <br><br>

      <label>Elevated password:<br>
        <input type="password" name="password" required>
      </label>
      <br><br>

      <label>Device list (one per line):<br>
        <textarea name="devices" placeholder="f5-01.example.com\nf5-02.example.com" required></textarea>
      </label>
      <br><br>

      <input type="submit" value="Run qkview-runner.sh">
    </form>

    {% if output %}
      <h2>Script output</h2>
      <pre>{{ output }}</pre>
    {% endif %}

  </body>
</html>
"""

# Path settings - adjust if you want different locations
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DEVICES_FILE = os.path.join(BASE_DIR, "devices.txt")
SCRIPT_PATH = os.path.join(BASE_DIR, "qkview-runner.sh")

@app.route("/", methods=["GET", "POST"])
def index():
    output = None

    if request.method == "POST":
        username = request.form.get("username", "").strip()
        password = request.form.get("password", "")
        devices_raw = request.form.get("devices", "").strip()

        # Normalize device list, remove empty lines and strip whitespace
        devices = [line.strip() for line in devices_raw.splitlines() if line.strip()]

        # Write devices to devices.txt (overwrite)
        try:
            with open(DEVICES_FILE, "w") as f:
                f.write("\n".join(devices) + ("\n" if devices else ""))
        except Exception as e:
            output = f"Failed to write devices file: {e}"
            return render_template_string(HTML, output=output)

        # Make sure script exists and is executable
        if not os.path.isfile(SCRIPT_PATH):
            output = f"Script not found: {SCRIPT_PATH}"
            return render_template_string(HTML, output=output)
        if not os.access(SCRIPT_PATH, os.X_OK):
            # try to chmod +x
            try:
                os.chmod(SCRIPT_PATH, 0o750)
            except Exception:
                pass

        # Build stdin for the script. The qkview-runner.sh prompts first for username then password.
        # Provide username\npassword\n so the script reads them as if typed interactively.
        stdin_data = f"{username}\n{password}\n"

        try:
            # Run the script and capture output (stdout + stderr)
            proc = subprocess.run(
                ["bash", SCRIPT_PATH],
                input=stdin_data,
                capture_output=True,
                text=True,
                timeout=60*60  # adjust timeout as needed (seconds)
            )
            # Combine stdout/stderr for display
            output = ""
            if proc.stdout:
                output += proc.stdout
            if proc.stderr:
                output += ("\n--- STDERR ---\n" + proc.stderr)
            output += f"\n\nExit code: {proc.returncode}"
        except subprocess.TimeoutExpired:
            output = "Script timed out."
        except Exception as e:
            output = f"Failed to run script: {e}"

    return render_template_string(HTML, output=output)


if __name__ == "__main__":
    # For development only. Use gunicorn for production as you already do.
    app.run(host="0.0.0.0", port=8000, debug=True)
```
