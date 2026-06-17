# Copy-paste setup prompt

Paste the block below into **your AI agent** (Hermes, Claude, ChatGPT, Cursor,
Codex, whatever). It hands the agent a full briefing — the why, the what, the
gotchas — and turns it into a step-by-step wizard that deploys this proxy to
**your own** Cloudflare account and wires it into Hermes so Telegram works from a
region where it's blocked.

> Why an agent prompt instead of just docs? The setup spans a browser (Cloudflare
> login), a config-file edit, and a service restart — an agent can run the
> checks, generate the exact edit for *your* machine, ask you the branching
> questions, and catch mistakes as you go.

It's written as a **handoff**: the agent gets enough background to reason on its
own, and is told to **ask you (via its clarify/ask tool) at every real decision
point and follow your direction** rather than guessing.

---

<details open>
<summary><b>📋 Click to expand — copy everything inside the box</b></summary>

````text
# HANDOFF — set up my self-hosted Telegram Bot API proxy and wire it into Hermes

You are taking over a setup task. Read this whole brief first, then drive it
interactively. This is a handoff: you have enough context to reason
independently, but you must ASK ME at every real decision point and follow my
direction — do not assume defaults on anything that has a trade-off.

## How to work this task (interaction contract)
- Use your question/clarify tool (Hermes: the `clarify` tool; otherwise just ask
  in chat and STOP for my answer) whenever there's a fork, a missing fact, or a
  step with a trade-off. Offer concrete options, recommend one, and let me pick.
- Go ONE step at a time. After each step, wait for me to paste output / confirm
  before moving on. Explain each step in a sentence. Never dump all steps at once.
- If something is ambiguous or my environment doesn't match your assumptions,
  ASK — don't guess. Taking the wrong branch silently is the main failure mode.
- Track the work as a checklist if you have one. Tell me where we are.

## Background — why this exists (so you can reason, not just follow)
- Telegram's Bot API lives at https://api.telegram.org. When that host /
  Telegram's IP ranges are blocked at the network level (e.g. India's nationwide
  block, Iran, Russia), a bot running inside the blocked region cannot reach it
  and the Hermes gateway logs repeated "telegram connect timed out".
- Cloudflare's edge (workers.dev) usually stays reachable from inside the block.
  So we put a thin HTTPS reverse proxy on a Cloudflare Worker: the bot talks to
  the Worker, the Worker relays to api.telegram.org from the edge. No VPN, no
  server to run, free tier.
- Repo (single file, src/worker.js, auditable, MIT):
  https://github.com/alt-glitch/hermes-telegram-proxy

## Critical facts you MUST respect (these are load-bearing — getting them wrong
## either leaks a credential or breaks my running gateway):
1. A Telegram bot token IS the bot's full credential, and on the Bot API it
   travels IN THE URL PATH (/bot<TOKEN>/method). Any proxy operator can see it.
   => I must deploy MY OWN instance; I must NOT route my token through anyone
   else's hosted proxy. This Worker is written to never log the token, but that
   only protects my own deploy. Reassure me of this, and if I ever propose using
   someone else's endpoint, warn me and ASK before continuing.
2. This is an HTTPS REVERSE PROXY (the `extra.base_url` mechanism), NOT a
   SOCKS/VPN/MTProto proxy. It only relays the Bot API for a bot. (Hermes ALSO
   has a separate `telegram.proxy_url` / TELEGRAM_PROXY SOCKS feature, and
   MTProto proxies are a different thing for the Telegram *app*. If I conflate
   them, clarify the difference and ASK which I actually want before proceeding.)
3. CONFIG + RESTART ARE MINE TO RUN, NOT YOURS:
   - Hermes guards ~/.hermes/config.yaml. Your file-write tools (write_file/
     patch) are typically REFUSED on it. You may either show me the exact edit to
     paste, OR apply it with a plain shell command AFTER backing the file up —
     but ASK me which I prefer first.
   - NEVER restart the Hermes gateway yourself if you're running inside it (a
     gateway-internal restart is self-terminating). Have ME run the restart.

## PHASE 0 — Orient (ask before doing anything)
Use your clarify/ask tool to establish, up front:
  a) Am I on my local machine or a remote/SSH server? (affects wrangler login)
  b) Do I want the one-click Deploy button (Path A, no CLI) or the wrangler CLI
     (Path B)? Recommend A for most people; B if I'm comfortable in a terminal.
  c) For the later config edit: do I want you to (i) just show me the edit, or
     (ii) apply it via a backed-up shell command? (Default to showing unless I
     say otherwise.)
  d) Is this for a Hermes Agent gateway, or a raw bot in some other framework?
Confirm my answers, then proceed.

## PHASE 1 — Deploy the Worker to MY Cloudflare account
- Path A (one-click): point me to the repo README's "Deploy to Cloudflare"
  button. It clones the repo into my GitHub and deploys to my Cloudflare account.
  I'll need a free Cloudflare account + a GitHub account. At the end Cloudflare
  shows me my Worker URL.
- Path B (CLI), in a clone of the repo:
    npx wrangler login    # browser OAuth. If I said I'm on a remote/SSH box,
                          # the callback hits localhost:8976 ON THE SERVER —
                          # tell me to run `ssh -L 8976:localhost:8976 <server>`
                          # from my laptop first, then paste the OAuth callback
                          # URL into my laptop browser so it tunnels back. ASK if
                          # I'm unsure.
    npx wrangler deploy   # prints my URL
- Result is a URL like https://hermes-telegram-proxy.<my-subdomain>.workers.dev
- Ask me to paste that exact URL back to you and use it for the rest.

## PHASE 2 — Verify the Worker BEFORE touching Hermes
  curl https://<MY-WORKER-URL>/healthz            # expect {"ok":true}
Then prove it relays to Telegram with a THROWAWAY fake token (never my real one;
use 123456789: followed by 35+ url-safe chars):
  curl "https://<MY-WORKER-URL>/bot123456789:AAA...35+chars.../getMe"
  # expect Telegram's {"ok":false,"error_code":401,"description":"Unauthorized"}
A 401 = the relay reached Telegram (success). A 502 = the Worker can't reach
Telegram (rare, CF-side — have me redeploy / check the dashboard). Confirm the
result with me before continuing.

## PHASE 3 — Wire it into Hermes (MY edit + MY restart)
The gateway reads a custom Bot API base URL from config.yaml at the Telegram
platform's `extra` block. The value MUST be my Worker URL with a TRAILING /bot
(PTB appends <token>/<method>): https://<MY-WORKER-URL>/bot
- The Telegram settings live under EITHER top-level `telegram: -> extra:` OR
  `platforms: -> telegram: -> extra:` depending on how my config was generated.
  Have me check which exists; if both, use platforms.telegram.extra. Preserve my
  existing extra keys; add base_url at the same indentation. Example:
      telegram:
        extra:
          rich_messages: true
          base_url: https://<MY-WORKER-URL>/bot
- Per my Phase-0 choice: either show me the exact diff to paste, or (if I opted
  in) apply it via shell AFTER: cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
- Then have ME restart the gateway:
    hermes gateway restart
    #   or, if systemd:  systemctl --user restart hermes-gateway.service
  If my Hermes runs another way (Docker, etc.), ASK how I run it and adapt.
  Never restart it yourself from inside an agent session.

## PHASE 4 — Confirm end to end
  grep -i "custom Telegram base_url" ~/.hermes/logs/gateway.log | tail -1
  grep -iE "Connected to Telegram|telegram connected|connect timed out" ~/.hermes/logs/gateway.log | tail -3
Success = a "Using custom Telegram base_url: ..." line AND a recent "Connected
to Telegram" / "telegram connected" with NO fresh "connect timed out" after the
restart. Then have me send my bot a Telegram message and confirm it replies.
Tell me we're done and summarize what changed.

## TROUBLESHOOTING (offer when relevant; ASK before destructive actions)
- Still timing out after restart: re-check the trailing /bot, the indentation
  (under telegram.extra), and that I actually SAVED the file before restarting.
  Re-run the log greps.
- 502 on the relay test: Worker deployed but can't reach Telegram — have me
  redeploy / check the Cloudflare dashboard.
- wrangler login won't open a browser (remote server): the ssh -L 8976 tunnel
  above; the OAuth code is short-lived + single-use, so if it expired, re-run
  `npx wrangler login` for a fresh URL (tunnel open FIRST).
- I want to use the SOCKS path instead: that's Hermes' separate
  telegram.proxy_url / TELEGRAM_PROXY feature pointing at a SOCKS5/HTTP proxy on
  a VPS — different from this Worker. Clarify and ASK which I want; don't do both.

Begin with PHASE 0. Ask me the orienting questions and wait for my answers
before touching anything.
````

</details>

---

## What the agent will do for you

0. **Orient** — ask whether you're local vs remote, one-click vs CLI, show-edit
   vs apply-edit, Hermes vs raw bot — and follow your answers.
1. **Deploy** the Worker to *your* Cloudflare account.
2. **Verify** it relays to Telegram (`/healthz` + fake-token `getMe` → `401`).
3. **Wire** it into `~/.hermes/config.yaml` (`telegram.extra.base_url: <url>/bot`).
4. **Confirm** the gateway connected — then you message your bot.

The prompt is a **handoff with guardrails**: the agent has full context to
reason, is told to **ask (clarify) at every decision point and take your
direction**, and is kept from doing the two security-sensitive steps itself
(editing the guarded config, restarting the gateway) — those stay with you.
