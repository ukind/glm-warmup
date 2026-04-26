# glm-warmup

Manipulate GLM Coding Plan's 5-hour usage window into resetting when you actually need it.

## Why

The GLM Coding Plan (Z.ai) gives you a prompt budget that resets every 5 hours. The window starts when you send your first prompt, and quota resets 5 hours after consumption.

So if you start working at 8:30 AM and burn through your budget by 11, you're locked out until your earliest consumed prompt was 5 hours ago. Dead hours in the middle of your day.

The fix is simple: fire a throwaway request before you start working. A GitHub Actions cron sends "hi" to `GLM-4.5-Air` (the cheapest model) at 6:15 AM. That starts the clock early. By the time you hit your limit, the first window has already reset — your next prompt anchors a fresh window through the afternoon.

Example schedule:
```markdown
            6am    7     8     9    10    11    12    1pm    2     3     4     5    6pm
             |     |     |     |     |     |     |     |     |     |     |     |     |

Before:                  [========== window 1 =========]
                          work ~8:30am-11am  ░░ dead ░░
                                                       [========== window 2 =========]
                                                                work ~1pm-6pm
          cron trigger
               │
               ▼
After:       [========== window 1 =========]
              ░░ idle ░░  work ~8:30am-11am
                                           [========== window 2 =========]
                                                   work ~11am-4pm
                                                                         [== win 3 ==]
                                                                         work ~4pm-6pm
```

## How it works

A scheduled GitHub Actions workflow sends a minimal `max_tokens: 1` request to the Coding Plan API using the cheapest available model. That single request is enough to start the 5-hour consumption clock. Cost-wise: one token on the cheapest model is nothing.

No CLI install needed — just a `curl` call to the OpenAI-compatible endpoint.

## Supported Services

| Service | Workflow File | Cheapest Model | Endpoint |
|---------|---------------|----------------|----------|
| GLM (Z.ai) | `warmup.yml` | `GLM-4.5-Air` | `api.z.ai` |
| OpenCode Go | `warmup-opencode-go.yml` | `MiniMax M2.5` | `opencode.ai` |
| CrofAI | `warmup-crofai.yml` | `kimi-k2.5` | `crof.ai` |
| Routing.run | `warmup-routing-run.yml` | `route/kimi-k2.5-highspeed` | `api.routing.run` |
| Ollama | `warmup-ollama.yml` | `glm-4.7` | `ollama.com` |

## Setup

### 1. Fork this repo

```bash
gh repo fork <your-username>/glm-warmup --clone
```

### 2. Get your API key

**For GLM (Z.ai):**
1. Go to [z.ai API Key management](https://z.ai/manage-apikey/apikey-list)
2. Create an API key
3. Make sure your GLM Coding Plan subscription is active

**For OpenCode Go:**
1. Go to [opencode.ai/auth](https://opencode.ai/auth) and subscribe to Go
2. Get your API key from the settings

### 3. Store it as a secret

**For GLM:**
```bash
cd glm-warmup
gh secret set ZAI_API_KEY
```

**For OpenCode Go:**
```bash
cd glm-warmup
**For OpenCode Go:**
```bash
cd glm-warmup
gh secret set OPENCODE_GO_API_KEY
```

**For CrofAI:**
```bash
cd glm-warmup
gh secret set CROFAI_API_KEY
```

**For Routing.run:**
```bash
cd glm-warmup
gh secret set ROUTING_RUN_API_KEY
```

**For Ollama:**
```bash
cd glm-warmup
gh secret set OLLAMA_API_KEY
```

Paste your key when prompted.

### 4. Set your schedule

Default is everyday at 7:00 AM WIB (UTC+7). To change it, edit the `cron` line in the workflow file:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'  # ← change this
```

That's a standard cron expression in UTC. Common conversions:

| Timezone | 7:00 AM local in UTC | Cron |
|---|---|---|
| US Pacific (UTC-7) | 2:00 PM | `0 14 * * *` |
| US Eastern (UTC-4) | 11:00 AM | `0 11 * * *` |
| US Central (UTC-5) | 12:00 PM | `0 12 * * *` |
| Central Europe (UTC+2) | 5:00 AM | `0 5 * * *` |
| Indonesia WIB (UTC+7) | 12:00 AM | `0 0 * * *` |
| India IST (UTC+5:30) | 1:30 AM | `30 1 * * *` |
| Japan JST (UTC+9) | 10:00 PM (prev) | `0 22 * * 0-6` |

Pick something 2-4 hours before you usually start working (depends on your usage pattern).

### 5. Test it

**For GLM:**
```bash
gh workflow run warmup.yml
```

**For OpenCode Go:**
```bash
gh workflow run warmup-opencode-go.yml
**For OpenCode Go:**
```bash
gh workflow run warmup-opencode-go.yml
```

**For CrofAI:**
```bash
gh workflow run warmup-crofai.yml
```

**For Routing.run:**
```bash
gh workflow run warmup-routing-run.yml
```

**For Ollama:**
```bash
gh workflow run warmup-ollama.yml
```

Check the logs. You should see `✓ Warmup complete — 5-hour window anchored`.

### 6. Verify

Next morning, check your usage statistics:
- GLM: [Z.ai Usage Statistics](https://z.ai/manage-apikey/usage)
- OpenCode Go: [OpenCode Go Usage](https://opencode.ai/usage)
- CrofAI: Check your [CrofAI dashboard](https://crof.ai)
- Routing.run: Check your [Routing.run dashboard](https://routing.run)
- Ollama: Check your [Ollama account settings](https://ollama.com)

You should see a tiny consumption from the warmup request, and your 5-hour window should be anchored to the cron time.

## About the Coding Plan quota

### GLM Coding Plan

From [Z.ai docs](https://docs.z.ai/devpack/overview):

- The 5-hour window is **dynamically refreshed** — quota resets 5 hours after consumption (not on fixed clock hours like Claude Code).
- There's also a separate **weekly cap** that resets every 7 days.
- The plan can only be used within supported coding tools (Claude Code, OpenCode, Cline, Cursor, etc.) — but the API endpoint works for warmup purposes.
- `GLM-4.5-Air` maps to Claude Code's "haiku" slot — the lightest, fastest model.

#### Plan limits

| Plan | 5-Hour Limit | Weekly Limit |
|---|---|---|
| Lite ($10/mo) | ~80 prompts | ~400 prompts |
| Pro ($30/mo) | ~400 prompts | ~2,000 prompts |
| Max ($90/mo) | ~1,600 prompts | ~8,000 prompts |

### OpenCode Go

From [OpenCode Go docs](https://dev.opencode.ai/docs/go/):

- The 5-hour window is **dynamically refreshed** — quota resets 5 hours after consumption.
- Monthly limits are also tracked.
- Most models use OpenAI-compatible endpoint (`@ai-sdk/openai-compatible`).
- `MiniMax M2.5` is the lightest model — 20,000 requests per 5 hours.

#### Plan limits

| Limit | Amount |
|-------|--------|
| 5-hour window | $12 (~$700 input + 52K cached + 150 output per request) |
| Weekly | $30 |
| Monthly | $60 |

## Questions

**Does this waste quota?** One request with `max_tokens: 1` on the cheapest model. You won't notice it.

**What if I'm already rate-limited?** Still works. The request reaches the servers either way, and it still starts the 5-hour consumption clock.

**Can I run this locally instead?** Yeah. Put this in a cron job or Task Scheduler:

```bash
# GLM
curl -s https://api.z.ai/api/coding/paas/v4/chat/completions \
  -H "Authorization: Bearer YOUR_ZAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"GLM-4.5-Air","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'

# OpenCode Go
curl -s https://opencode.ai/zen/go/v1/chat/completions \
  -H "Authorization: Bearer YOUR_OPENCODE_GO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"MiniMax-M2.5","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'

# CrofAI
curl -s https://crof.ai/v1/chat/completions \
  -H "Authorization: Bearer YOUR_CROFAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'

# Routing.run
curl -s https://api.routing.run/v1/chat/completions \
  -H "Authorization: Bearer YOUR_ROUTING_RUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"route/kimi-k2.5-highspeed","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'

# Ollama
curl -s https://ollama.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_OLLAMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"glm-4.7","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'
```

GitHub Actions is just easier because your machine doesn't need to be awake.

**API key security?** The key is stored as a GitHub Actions secret — it's encrypted and only exposed to the workflow at runtime. Never commit it to the repo.

## License

MIT