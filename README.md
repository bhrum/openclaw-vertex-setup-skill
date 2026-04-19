# OpenClaw Vertex Setup Skill

This repository contains a Codex/OpenClaw skill for configuring OpenClaw to use Google Vertex AI Gemini models through the normal startup path.

It is designed to make these flows work together:

- `openclaw models status`
- `openclaw gateway start` / `openclaw gateway restart`
- `openclaw tui`
- launchd-managed Gateway startup on macOS

It also documents how to avoid a common failure mode where the globally installed npm `openclaw` CLI and the installed Gateway service come from different versions.

## What This Skill Covers

- Google Cloud `gcloud` ADC login
- `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION`
- shared OpenClaw env via `~/.openclaw/.env`
- default model setup in `~/.openclaw/openclaw.json`
- practical `google-vertex` auth-profile repair
- Gateway service install, restart, and verification
- end-to-end testing through both local agent mode and Gateway/TUI mode

## Current Model Guidance

This repo currently uses:

- OpenClaw config model id: `google-vertex/gemini-3.1-pro-preview`

Important distinction:

- `gemini-3.1-pro-preview` is the newer preview track
- `gemini-2.5-pro` is still the latest stable Vertex Pro model

So if you want the newest preview path, use `3.1-pro-preview`.
If you want the safer production fallback, use `2.5-pro`.

## Quick Start

1. Log in with ADC:

```bash
gcloud auth application-default login
gcloud config set project <project-id>
gcloud auth application-default set-quota-project <project-id>
```

2. Put shared Vertex env into `~/.openclaw/.env`:

```dotenv
GOOGLE_CLOUD_PROJECT=<project-id>
GOOGLE_CLOUD_LOCATION=global
GOOGLE_APPLICATION_CREDENTIALS=/Users/<user>/.config/gcloud/application_default_credentials.json
```

3. Set the default model in `~/.openclaw/openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "google-vertex/gemini-3.1-pro-preview"
      }
    }
  }
}
```

4. Reinstall the Gateway service from the active CLI:

```bash
openclaw gateway install --force
openclaw gateway restart
openclaw gateway status --deep
```

5. Test the Gateway path:

```bash
openclaw agent --agent main --message "Reply with exactly: OPENCLAW_VERTEX_OK" --json
openclaw tui
```

## Upgrade Rule

After every:

```bash
npm install -g openclaw@latest
```

run:

```bash
openclaw gateway install --force
openclaw gateway restart
openclaw gateway status --deep
```

Why this matters:

- the CLI may update
- the old LaunchAgent may still point to a previous install
- mixed versions can cause TUI/Gateway protocol errors such as `invalid connect params`

## Known Limitation

As tested on 2026-04-19, `google-vertex/gemini-3.1-pro-preview` can still produce incomplete turns in the current OpenClaw execution path even when auth and Gateway wiring are correct.

If that happens, keep the environment and Gateway fixes, and temporarily switch the default model back to:

```json
"google-vertex/gemini-2.5-pro"
```

That gives you the stable Vertex fallback while keeping the rest of the integration intact.

## Repository Layout

- [SKILL.md](./SKILL.md): machine-facing skill instructions
- [agents/openai.yaml](./agents/openai.yaml): skill metadata for UI integration

## Repository

- GitHub: [bhrum/openclaw-vertex-setup-skill](https://github.com/bhrum/openclaw-vertex-setup-skill)
