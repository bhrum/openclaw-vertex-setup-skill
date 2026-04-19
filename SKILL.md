---
name: openclaw-vertex-setup
description: Configure OpenClaw to use Google Vertex AI Gemini models so normal OpenClaw startup, Gateway service startup, and TUI usage all work without manual shell setup. Covers gcloud ADC login, project and location setup, OpenClaw model defaults, shared env wiring, auth profile fixes, and end-to-end verification.
---

# OpenClaw Vertex Setup

Use this skill when the task is to configure or repair OpenClaw so it uses Vertex AI models such as `google-vertex/gemini-2.5-pro`, and the result must work from normal `openclaw` commands instead of only from a manually prepared shell.

Target outcome:

- `openclaw models status` recognizes Vertex
- `openclaw gateway start` or `openclaw gateway restart` works
- `openclaw tui` can chat through Vertex without manually exporting env vars first

Keep the workflow minimal and verify each layer separately:

1. Vertex itself
2. OpenClaw model discovery
3. OpenClaw local agent execution
4. Gateway and TUI execution

## What to inspect first

- User config: `~/.openclaw/openclaw.json`
- Shared OpenClaw env: `~/.openclaw/.env`
- Agent auth store: `~/.openclaw/agents/main/agent/auth-profiles.json`
- Shell env: `~/.zshrc`
- ADC file: `~/.config/gcloud/application_default_credentials.json`

Confirm the default OpenClaw model points at Vertex:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "google-vertex/gemini-2.5-pro"
      }
    }
  }
}
```

## Required environment

Ensure these are available to the process that launches the Gateway:

```bash
export GOOGLE_CLOUD_PROJECT="<project-id>"
export GOOGLE_CLOUD_LOCATION="global"
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json"
```

ADC must exist at the standard path, or `GOOGLE_APPLICATION_CREDENTIALS` must point to it:

```bash
gcloud auth application-default login
gcloud config set project <project-id>
gcloud auth application-default set-quota-project <project-id>
```

If `gcloud` cannot be installed via `dl.google.com`, the Google Cloud CLI tarball can be downloaded from `storage.googleapis.com/cloud-sdk-release/`.

## Make normal startup work

Do not rely only on the interactive shell. Put the Vertex env into OpenClaw's shared env file so launchd/systemd and plain `openclaw` commands see the same values:

`~/.openclaw/.env`

Example:

```dotenv
GOOGLE_CLOUD_PROJECT=<project-id>
GOOGLE_CLOUD_LOCATION=global
GOOGLE_APPLICATION_CREDENTIALS=/Users/<user>/.config/gcloud/application_default_credentials.json
```

This is the most reliable fix for:

- `openclaw gateway start`
- `openclaw gateway restart`
- `openclaw tui`
- launchd/systemd service startup

## OpenClaw config changes

Recommended minimum `~/.openclaw/openclaw.json` changes:

- Set default model to `google-vertex/gemini-2.5-pro`
- Enable shell env import:

```json
{
  "env": {
    "shellEnv": {
      "enabled": true,
      "timeoutMs": 15000
    }
  }
}
```

## Source-level hardening

If you are repairing OpenClaw itself rather than only configuring a machine, make these code-level fixes in the OpenClaw repo:

- add `GOOGLE_CLOUD_PROJECT`, `GCLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, and `GOOGLE_APPLICATION_CREDENTIALS` to the shell env fallback key list
- make Vertex auth detection use a stable ADC file check instead of a cold-start-sensitive provider helper
- pass Vertex env vars through daemon service environment generation

These changes prevent the common failure where `openclaw agent --local` works but Gateway or TUI fails because the service process started without Vertex project/location env.

## Auth profile fix

OpenClaw can report `No API key found for provider "google-vertex"` even when ADC is already valid. If `openclaw models status` shows `google-vertex` under `Missing auth`, add a profile entry in:

`~/.openclaw/agents/main/agent/auth-profiles.json`

Example:

```json
{
  "profiles": {
    "google-vertex:default": {
      "type": "api_key",
      "provider": "google-vertex",
      "key": "<authenticated>"
    }
  },
  "lastGood": {
    "google-vertex": "google-vertex:default"
  }
}
```

This is a practical compatibility fix. Prefer the source-level hardening above when you control the OpenClaw codebase.

## Verification sequence

Run these in order.

### 1. Verify ADC token

```bash
gcloud auth application-default print-access-token | head -c 20; echo
```

### 2. Verify direct Vertex REST call

Use the global endpoint and a tiny prompt:

```bash
python3 - <<'PY'
import json, subprocess, urllib.request
project='<project-id>'
model='gemini-2.5-pro'
url=f'https://aiplatform.googleapis.com/v1/projects/{project}/locations/global/publishers/google/models/{model}:generateContent'
token=subprocess.check_output(['zsh','-lc','gcloud auth application-default print-access-token'], text=True).strip()
payload={
  'contents':[{'role':'user','parts':[{'text':'Reply with exactly: VERTEX_OK'}]}],
  'generationConfig':{'temperature':0}
}
req=urllib.request.Request(url, data=json.dumps(payload).encode(), method='POST')
req.add_header('Authorization', f'Bearer {token}')
req.add_header('Content-Type', 'application/json')
with urllib.request.urlopen(req, timeout=60) as r:
    print(r.status)
    print(r.read().decode())
PY
```

### 3. Verify OpenClaw sees the model

```bash
openclaw models list
openclaw models status
```

Expected signals:

- `google-vertex/gemini-2.5-pro` appears in model list
- auth overview shows `source=gcloud adc`

### 4. Verify local OpenClaw agent

```bash
source ~/.zshrc >/dev/null 2>&1
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json"
openclaw agent --local --agent main --message "Reply with exactly: OPENCLAW_VERTEX_OK" --json
```

### 5. Verify Gateway/TUI path

Install the Gateway service if it is not loaded:

```bash
openclaw gateway install --force
openclaw gateway start
openclaw gateway status --deep
```

If you are only doing a foreground smoke test, start the gateway directly:

```bash
openclaw gateway run --verbose
```

In another terminal:

```bash
openclaw tui
```

Also verify the Gateway path directly:

```bash
openclaw agent --agent main --message "Reply with exactly: OPENCLAW_VERTEX_OK" --json
```

## Common failures

### `openclaw: command not found`

Install the local repo as a global command:

```bash
cd ~/openclaw
npm link
```

### TUI shows `(no output)`

Check the session `.jsonl` log. If the assistant message contains:

`Vertex AI requires a project ID. Set GOOGLE_CLOUD_PROJECT/GCLOUD_PROJECT or pass project in options.`

then the Gateway process was started without the required Vertex env. Restart the Gateway from a shell that has:

- `GOOGLE_CLOUD_PROJECT`
- `GOOGLE_CLOUD_LOCATION`
- optionally `GOOGLE_APPLICATION_CREDENTIALS`

Then move those values into `~/.openclaw/.env` so the next normal service start inherits them.

### `No API key found for provider "google-vertex"`

This usually means OpenClaw's auth profile store is missing a `google-vertex` profile entry, even though ADC is already valid. Patch `auth-profiles.json` as described above, then re-run:

```bash
openclaw models status
```

### `gateway restart` says service not loaded

Install the service first:

```bash
openclaw gateway install --force
openclaw gateway start
```

Then confirm with:

```bash
openclaw gateway status --deep
```

## Success criteria

The setup is complete when all of these are true:

- direct Vertex REST call returns `200`
- `openclaw models status` shows `google-vertex`
- `openclaw agent --local` returns `OPENCLAW_VERTEX_OK`
- `openclaw agent` via Gateway returns `OPENCLAW_VERTEX_OK`
- `openclaw gateway status --deep` shows `RPC probe: ok`
- `openclaw tui` can answer a prompt through `google-vertex/gemini-2.5-pro`
