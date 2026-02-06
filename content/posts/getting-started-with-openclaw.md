---
title: "Getting Started with OpenClaw on DigitalOcean"
date: 2026-02-05
draft: false
tags: ["agentic-systems", "self-hosting", "telegram", "claude"]
description: "Setting up an OpenClaw agent harness on DigitalOcean's one-click VPS, with Telegram as the interface channel."
---

I've been wanting to experiment with agentic harnesses for a while, and OpenClaw seemed like a reasonable starting point. That said, the project has a rocky security history that shaped how I approached the deployment.

Late last  year, ClawdBot (an OpenClaw-based agent) shipped with exposed credentials and insecure defaults. Attackers found the deployment, gained access, and hijacked it—retrieving API keys, private chat logs, and in some cases driving the agent to take actions against connected services. Not great. Given that context, I wanted a deployment path that minimized my own configuration surface area, at least initially.

## Why the DigitalOcean one-click image

DigitalOcean offers a [one-click VPS image for OpenClaw](https://marketplace.digitalocean.com/apps/openclaw) at $24/month, which gets you 2GB of memory, a 90GB NVMe SSD, and 3TB of transfer. More importantly, it comes with OpenClaw 2026.2.1 pre-installed and reasonably locked down out of the box. I figured paying a bit more for a managed starting point was worth avoiding the footguns that come with manual installation.

## Initial setup

After spinning up the droplet, I SSH'd in and ran through the automated setup flow. The main thing it needs is an LLM API key—I grabbed mine from the [Claude Console](https://platform.claude.com/settings/keys) and pasted it when prompted. The setup also offers a manual configuration path for things like rate limits, model selection, and tool permissions. I skipped that for now; I wanted to get something running first and come back to hardening later.

To verify the installation worked, I ran:

```bash
/opt/openclaw-tui.sh
```

This launches the terminal UI. Everything looked healthy, so I bookmarked the dashboard URL it printed and moved on to setting up a messaging interface.

## Adding Telegram as a channel

OpenClaw supports several channel types for interacting with your agent. I went with Telegram over WhatsApp for a practical reason: Telegram lets you create a bot token through @BotFather that's completely separate from your personal phone number. WhatsApp, by contrast, requires either your real number or a separate phone with an eSIM, which felt like unnecessary friction for a first experiment.

Adding the channel started with:

```bash
/opt/openclaw-cli.sh channels add
```

I selected "Telegram (BotAPI)" from the menu. The CLI scaffolds out the channel config, but I couldn't get the bot to actually connect until I edited the JSON directly through the config tab in the OpenClaw Gateway Dashboard. The working configuration looked like this:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "botToken": "YOUR_BOT_TOKEN_HERE",
    "allowFrom": ["YOUR_USER_ID_HERE"]
  }
}
```

The `allowFrom` field restricts which Telegram user IDs can interact with the bot—worth setting explicitly given OpenClaw's history with unauthorized access.

After saving that config, the bot came online and I could message it from Telegram. The whole process took maybe thirty minutes, most of which was waiting for the droplet to provision and figuring out why the CLI-generated config wasn't sufficient.

## What's next

I have more work to do here. The manual configuration I skipped includes things like tool sandboxing and request logging that I'll want before connecting the agent to anything sensitive. But as a first pass at running an agentic harness, this was straightforward enough to be encouraging.
