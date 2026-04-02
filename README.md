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

A scheduled GitHub Actions workflow sends a minimal `max_tokens: 1` request to the GLM Coding Plan API using `GLM-4.5-Air` (the lightest, fastest model). That single request is enough to start the 5-hour consumption clock. Cost-wise: one token on the cheapest model is nothing.

No CLI install needed — just a `curl` call to Z.ai's OpenAI-compatible endpoint.

## Setup

### 1. Fork this repo

```bash
gh repo fork <your-username>/glm-warmup --clone
```

### 2. Get your Z.ai API key

1. Go to [z.ai API Key management](https://z.ai/manage-apikey/apikey-list)
2. Create an API key
3. Make sure your GLM Coding Plan subscription is active

### 3. Store it as a secret

```bash
cd glm-warmup
gh secret set ZAI_API_KEY
```

Paste your key when prompted.

### 4. Set your schedule

Default is weekdays at 9:15 AM UTC. Change it with a repo variable:

```bash
    gh variable set WARMUP_CRON --body "15 11 * * 1-5"
```

That's a standard cron expression in UTC. Common conversions:

| Timezone | 6:15 AM local in UTC | Cron |
|---|---|---|
| US Pacific (UTC-7) | 1:15 PM | `15 13 * * 1-5` |
| US Eastern (UTC-4) | 10:15 AM | `15 10 * * 1-5` |
| US Central (UTC-5) | 11:15 AM | `15 11 * * 1-5` |
| Central Europe (UTC+2) | 4:15 AM | `15 4 * * 1-5` |
| Indonesia WIB (UTC+7) | 11:15 PM prev day | `15 23 * * 0-4` |
| India IST (UTC+5:30) | 12:45 AM | `45 0 * * 1-5` |
| Japan JST (UTC+9) | 9:15 PM prev day | `15 21 * * 0-4` |
| Australia AEST (UTC+10) | 8:15 PM prev day | `15 20 * * 0-4` |

Pick something 2-4 hours before you usually start working (depends on your usage pattern).

### 5. Test it

```bash
gh workflow run warmup.yml
```

Check the logs. You should see `✓ Warmup complete — 5-hour window anchored`.

### 6. Verify

Next morning, check your [Z.ai Usage Statistics](https://z.ai/manage-apikey/usage). You should see a tiny consumption from the warmup request, and your 5-hour window should be anchored to the cron time.

## About the GLM Coding Plan quota

From [Z.ai docs](https://docs.z.ai/devpack/overview):

- The 5-hour window is **dynamically refreshed** — quota resets 5 hours after consumption (not on fixed clock hours like Claude Code).
- There's also a separate **weekly cap** that resets every 7 days.
- The plan can only be used within supported coding tools (Claude Code, OpenCode, Cline, Cursor, etc.) — but the API endpoint works for warmup purposes.
- `GLM-4.5-Air` maps to Claude Code's "haiku" slot — the lightest, fastest model.

### Plan limits

| Plan | 5-Hour Limit | Weekly Limit |
|---|---|---|
| Lite ($10/mo) | ~80 prompts | ~400 prompts |
| Pro ($30/mo) | ~400 prompts | ~2,000 prompts |
| Max ($90/mo) | ~1,600 prompts | ~8,000 prompts |

## Questions

**Does this waste quota?** One request with `max_tokens: 1` on GLM-4.5-Air. You won't notice it.

**What if I'm already rate-limited?** Still works. The request reaches Z.ai's servers either way, and it still starts the 5-hour consumption clock.

**Can I run this locally instead?** Yeah. Put this in a cron job or Task Scheduler:

```bash
curl -s https://api.z.ai/api/coding/paas/v4/chat/completions \
  -H "Authorization: Bearer YOUR_ZAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"GLM-4.5-Air","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'
```

GitHub Actions is just easier because your machine doesn't need to be awake.

**API key security?** The key is stored as a GitHub Actions secret — it's encrypted and only exposed to the workflow at runtime. Never commit it to the repo.

## License

MIT
