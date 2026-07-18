# Portfolio Digest & Live Chat Assistant

Two connected automations built around one goal: turning scattered market-checking into genuine financial literacy. A **scheduled daily digest** delivers an AI-enriched market briefing every day at close. A **live conversational bot** answers on-demand questions about the market and your own positions, in plain English, with real memory across a conversation.

Zero manual steps. Zero subscription cost beyond pennies of API usage. Built with n8n, JavaScript, Discord.js, the Finnhub API, and the Anthropic API.

![Automation canvas](screenshots/canvas.png)

## The Problem

I wanted two different things that most tools force you to choose between: a passive daily brief that tells me what happened and why, and an active tool I could actually ask follow-up questions — "how far is this from its 52-week high," "how much have I made on this position" — without opening five apps or doing math by hand.

## Part 1: The Daily Digest

Every day at market close, a scheduled n8n workflow:

1. Pulls live quotes for 30 tickers (stocks + crypto) from Finnhub, spanning personal holdings and a themed, sector-tagged watchlist
2. Computes day-over-day change against **its own self-maintained price history** — no paid historical-data API needed
3. Pulls 52-week high/low data and flags any ticker within 10% of either extreme, with a visual range gauge for holdings
4. Tracks real dollar-impact and gain/loss-since-purchase using actual cost basis
5. Computes a volatility-ratio signal — how today's move compares to a ticker's own typical daily swing, self-activating once ~2 weeks of history accumulates
6. Identifies the day's top 5 movers and fetches recent headlines for each
7. Sends those headlines to Claude, which writes a grounded narrative explaining *why* each mover likely moved — explicitly instructed to say "no clear catalyst found" rather than fabricate a reason
8. Formats everything into three purpose-built Discord messages (Top Stories, Holdings, Watching) so nothing gets truncated
9. Saves today's prices as a dated snapshot for tomorrow's comparison

**Daily Portfolio Digest:**

![Portfolio digest](screenshots/portfolio-digest.png)

**Top Stories, AI-generated:**

![Top Stories digest](screenshots/top-stories.png)

## Part 2: The Live Chat Assistant

A second, independent system that listens for questions in Discord and answers them live, using fresh data — not the daily snapshot.

**Architecture:**

```
Discord message ("!ask ...")
        │
Bridge script (Node.js + discord.js)
   — listens for messages, filters for the !ask prefix,
     forwards the question to n8n via webhook
        │
n8n Webhook Trigger
        │
Parse Question ─── extracts ticker (word-boundary matched
        │           against aliases), detects personal-portfolio
        │           intent, falls back to the last-mentioned
        │           ticker in conversation history if none found
        │
   ┌────┼──────────┬──────────────┬─────────────┐
   │    │          │              │             │
Live   Live      Live          Check         Load
Quote  News      Metrics       Positions     History
   │    │          │              │             │
   └────┴──────────┴──────────────┴─────────────┘
                    │
                  Merge (sync gate, 5 inputs)
                    │
              Build Chat Prompt ─── weaves live data,
                    │               position data, and recent
                    │               conversation into one prompt
                  Call Claude
                    │
              Save History ─── appends this exchange
                    │           (with symbol) to a local file
              Send Reply
                    │
              Discord webhook
```

**What it can answer:**
- Live price, day change, and recent headlines for any tracked ticker
- 52-week high/low and Finnhub's own computed 5-day, 3-month, 6-month, and 1-year returns
- Personal position questions — real shares, cost basis, current value, and gain/loss, read from a shared positions file
- Genuine follow-up questions — "what about compared to last month" correctly infers the ticker from the previous exchange, using a rolling conversation history that resets after an hour of inactivity

**Design decisions worth explaining:**

**Word-boundary ticker matching, not substring matching.** An early version matched ticker aliases with plain substring search, which meant asking "why did GLW move so *much*" accidentally matched the ticker `MU` — because "much" contains "mu." Fixed with regex word-boundaries so short tickers only match as standalone words.

**Conversation memory via a local history file, not a stateful LLM session.** Each exchange is saved with its question, answer, matched ticker, and timestamp. Future prompts include recent exchanges as context, and a fresh prompt is still built from scratch each time — Claude has no built-in memory; the illusion of conversation comes entirely from what we choose to feed it.

**Symbol inheritance for context-free follow-ups.** If a message has no ticker in it at all, `parse question` checks conversation history (within the last hour) for the most recently discussed symbol before giving up — this is what makes "what about last week" work without repeating the ticker name.

**A dedicated bridge process, because n8n can't listen.** n8n's Discord integration is send-only — there's no native "message received" trigger. A small standalone Node.js script using `discord.js` handles the actual listening, filters for a `!ask` prefix, and forwards questions to n8n via webhook. This is a second always-running process, separate from n8n itself.

**Shared, file-based portfolio data, not duplicated config.** Position data (shares, cost basis) lives in one JSON file that both the digest and the chat bot can read, rather than being hardcoded separately in each — one place to update when a position changes.

## What Broke and How I Fixed It

**From the digest build:**
- Safari secure-cookie block, file-access restrictions, sandboxed `fs` module — resolved with environment flags, baked into a launch script
- Silent snapshot corruption from a misplaced file-write step, diagnosed by reading item counts on the canvas
- A timezone bug where UTC-based date logic silently rolled to the next calendar day after ~8pm Eastern, corrupting comparisons
- Double-execution from two branches feeding one node directly — fixed with a Merge node as a pure synchronization gate
- Discord's 2000-character message limit — solved with a three-message split rather than truncating content

**From the chat bot build:**
- **The substring-matching ticker bug** described above (GLW question answered about MU instead)
- **Zombie background processes.** Multiple terminal-tab freezes during development left old `node bridge.js` processes running invisibly in the background, competing for the same Discord bot token and silently intercepting messages meant for the active session. Diagnosed with `ps aux | grep` and resolved with `kill`.
- **Test vs. Production webhook confusion.** n8n's test webhook URL only fires once per manual arm-click; a `sed` replacement meant to switch to the Production URL silently failed to match, so the bridge kept sending to the test endpoint for longer than expected. Fixed by rewriting the file directly rather than trusting an in-place text substitution.
- **Reliability gaps closed with persistence tooling.** Both n8n and the bridge script now run under macOS LaunchAgents (`RunAtLoad`, and `KeepAlive` for the bridge specifically, so it self-restarts if it ever crashes) rather than depending on a terminal window staying open. n8n additionally runs wrapped in `caffeinate -i` to prevent the Mac from idle-sleeping mid-session, paired with a system Energy setting to prevent sleep while plugged in.

## Stack

| Piece | Choice | Why |
|---|---|---|
| Orchestration | n8n (self-hosted, local) | Free, visual debugging, per-item execution model |
| Live listening | Node.js + discord.js (bridge script) | n8n has no native Discord message trigger |
| Market data | Finnhub free tier | Quotes, 52-week metrics, company news — one key, one provider |
| AI synthesis | Anthropic API (Claude) | Structured output, strong instruction-following for grounding rules |
| Persistence | Dated JSON snapshots + a conversation-history file + a shared positions file | Free historical data and memory via self-maintained state, no database needed |
| Delivery | Discord webhooks | No auth, no OAuth, instant mobile push |
| Reliability | macOS LaunchAgents + `caffeinate` | Both systems survive sleep, terminal closures, and crashes without manual intervention |

## Running It

**Daily digest:**
1. Import `digest-workflow.json` into n8n
2. Add Finnhub and Anthropic API keys to the relevant nodes
3. Create a Discord webhook and paste the URL into the Discord node
4. Edit the ticker array and the shared positions file to match your own watchlist and holdings
5. Launch n8n with the required environment flags (see `start-n8n.sh`), publish the workflow

**Live chat assistant:**
1. Register a Discord Bot Application, enable the Message Content privileged intent, and invite it to your server
2. Import `chat-workflow.json` into n8n, copy its webhook's Production URL
3. Clone the bridge script folder, `npm install discord.js`, add your bot token and the webhook URL to `bridge.js`
4. Run `node bridge.js`, publish the n8n chat workflow
5. Ask anything in Discord with a `!ask` prefix

## Known Limitations

- No true historical daily-candle data — only Finnhub's pre-computed return summaries (5-day/13-week/26-week/52-week), since real historical candles are a paid Finnhub tier
- Chat is scoped to tracked tickers only; asking about an untracked stock won't resolve to a symbol
- No multi-ticker comparison logic — "compare NVDA and AMD" only resolves the first match
- Crypto news coverage is thin, since Finnhub's company-news endpoint isn't built for crypto assets
- Conversation memory is single-user by design; the history file has no per-user separation

## Roadmap

- Whole-market access beyond the tracked ticker list
- Thematic/sector questions ("how's the AI chip industry doing") reasoning across multiple tickers at once
- Concentration risk alerts and earnings calendar warnings
- Migrating off local hardware entirely (a small cloud instance or Raspberry Pi) to remove any dependency on the Mac being awake

---

*Built July 2026. Not investment advice — this tool reports what happened, where the evidence supports it and why. The thinking is still my job.*
