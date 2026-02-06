---
title: "Hardening OpenClaw: Closing the Gaps in the One-Click Deploy"
date: 2026-02-06
description: "The security work I skipped during initial setup—locking down the dashboard, configuring Caddy, and running the audit."
tags: ["self-hosting", "security", "ai", "vps"]
draft: false
---

In my last post, I mentioned skipping the manual configuration and planning to "come back to hardening later." This is later.

I compared my running setup against Grant Harvey's security guide for OpenClaw and found that while DigitalOcean's one-click image made some good choices, it also left the dashboard exposed to the public internet. Given OpenClaw's history with the ClawdBot incident, that felt worth fixing before I connected anything else.

## What was already fine

The one-click deploy did get some things right. The OpenClaw service runs under a dedicated `openclaw` user rather than root—I verified this with `ps aux | grep openclaw`. If the agent hallucinates a destructive command or gets prompt-injected, the blast radius is limited to what that user can access.

Caddy handles HTTPS with automatic Let's Encrypt certificates, so traffic between my browser and the VPS is encrypted. And the gateway requires a token for dashboard access, which provides a baseline of protection.

## The problem: dashboard on the public internet

The issue is that anyone with the dashboard URL and token can access my dashboard. The token helps, but tokens in URLs get logged by browsers, synced to history, and occasionally pasted into the wrong chat window. The secure approach is to keep the dashboard off the public internet entirely and access it through an SSH tunnel.

This creates a tension with Telegram. The bot needs to receive webhooks from Telegram's servers, which means some routes have to stay publicly accessible. The fix is to configure Caddy to allow webhook traffic while blocking dashboard routes.

## Reconfiguring Caddy

The original Caddyfile proxied everything to the OpenClaw gateway:

```bash
YOUR_DROPLET_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

I modified it to return a 404 for dashboard paths:

```bash
YOUR_DROPLET_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    
    @dashboard {
        path / /chat* /settings* /ui*
    }
    respond @dashboard "Not Found" 404
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

After `systemctl reload caddy`, external services can still hit `/webhook/*` and `/api/*` endpoints, but requests to `/chat` or the root path get a 404. Telegram keeps working; the dashboard doesn't.

## Accessing the dashboard via SSH tunnel

Now I reach the dashboard through an SSH tunnel:

```bash
ssh -L 18789:localhost:18789 user@YOUR_DROPLET_IP
```

Then open `http://localhost:18789?token=YOUR_GATEWAY_TOKEN` in my browser. The connection routes through SSH, encrypted and authenticated by my key, and the dashboard never touches the public internet. When I close the terminal, the tunnel closes too.

One thing that tripped me up: the security guide I was following used port 3000 in its examples, but OpenClaw's gateway actually runs on port 18789. I spent a few minutes getting "connection refused" errors before checking `/opt/openclaw.env` and realizing the mismatch.

## Other cleanup

The one-click image runs a setup wizard on every SSH login, even after initial configuration. The culprit was two lines at the bottom of `/root/.bashrc`:

```bash
chmod +x /etc/setup_wizard.sh
/etc/setup_wizard.sh
```

I removed them. In the process, I managed to corrupt the bashrc file and had to restore it from `/etc/skel/.bashrc`. Not my finest moment, but easily fixed.

I also ran the built-in security audit:

```bash
/opt/openclaw-cli.sh security audit --fix
```

It passed with no critical issues—just fixed some file permissions automatically. The audit output includes an "attack surface summary" that's worth reviewing if you've enabled tools or browser control.

## What's left

I'm still logging in as root for administration, which isn't ideal. The `openclaw` user exists and the service runs under it, but I haven't set up SSH key access for that user yet. I should also set spending caps in the Anthropic dashboard to prevent runaway costs if something goes wrong.

But the main gap—dashboard exposed to the internet—is closed. The bot still works through Telegram, and accessing the control interface now requires my SSH key rather than just a URL that could leak.
