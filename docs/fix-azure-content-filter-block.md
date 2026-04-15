# Fix: Azure Content Filter Blocking Hermes Agent API Calls

**Date:** 2026-04-07
**Affected Server:** 192.168.150.37 (p16s)
**Affected Model:** grok-4-1-fast-reasoning via LiteLLM proxy (192.168.150.100:4004)
**Status:** Resolved

---

## Symptom

Hermes Agent on .37 fails with HTTP 431 or HTTP 400 errors when making API calls through LiteLLM to the Azure Foundry grok-4-1-fast-reasoning model:

```
APIStatusError [HTTP 431]
litellm.APIError: AzureException APIError - {"error":{"code":"Unknown Status Code","message":"","status":431}}
```

Or:

```
litellm.BadRequestError: AzureException - {"error":{"message":"The response was filtered due to the prompt
triggering Microsoft's content management policy.","type":null,"param":"prompt","code":"content_filter","status":400}}
```

The error is **non-retryable** (HTTP 4xx). Hermes correctly identifies it as non-retryable but cannot recover.

Simple API calls (e.g., via curl with short prompts) succeed. Only Hermes's full system prompt triggers the block.

## Root Cause

### Azure Default Safety Policies

Azure AI Foundry applies **default content filtering** to all model deployments, even when no custom content filters are configured. These defaults include:

| Filter | Type | Default State |
|--------|------|---------------|
| Hate and Fairness | Severity (Low/Medium/High) | Block at Medium+ |
| Violence | Severity | Block at Medium+ |
| Sexual | Severity | Block at Medium+ |
| Self-Harm | Severity | Block at Medium+ |
| **Jailbreak (Prompt Shield)** | **Binary (on/off)** | **ON** |
| Protected Material - Text | Binary | ON |
| Protected Material - Code | Binary | ON |

The **Jailbreak Prompt Shield** is a contextual classifier that scans all input prompts for jailbreak patterns. It is enabled by default on all deployments and operates independently of any custom content filter configuration.

### Trigger Content in Hermes System Prompt

Hermes Agent injects an `<available_skills>` XML block into every system prompt, listing all installed skills with their descriptions. Two skills contained descriptions that triggered Azure's Jailbreak Prompt Shield:

1. **`godmode`** (skill path: `skills/red-teaming/godmode/`)
   - Description: *"Jailbreak API-served LLMs using G0DM0D3 techniques"*
   - Trigger words: `Jailbreak`, `G0DM0D3`, `safety-bypass`, `red-teaming`

2. **`obliteratus`** (skill path: `skills/mlops/inference/obliteratus/`)
   - Description: *"Remove refusal behaviors from open-weight LLMs"*
   - Trigger words: `Remove refusal`, `uncensoring`, `guardrails`

These descriptions appear in every API call's system prompt (~15KB), combined with agent autonomy language and tool definitions (~47KB total payload). Azure's classifier evaluates the full context, not individual keywords, and flags the combined content as a jailbreak attempt.

### Why Curl Tests Passed

Simple curl tests with short system prompts succeed because:
- The payload is small (~200 bytes vs ~47KB)
- No jailbreak-associated language is present
- The Prompt Shield classifier needs sufficient context to trigger

## Security Assessment of Removed Skills

### godmode - HIGH RISK

An active jailbreak toolkit containing:
- 33 input obfuscation techniques to evade safety classifiers
- Ready-to-use jailbreak templates targeting Claude, GPT, Gemini, Grok
- `auto_jailbreak()` function that modifies `config.yaml` and creates `prefill.json` to persist jailbreak state across all future API calls
- Multi-model racing to find least-filtered responses
- Scoring system optimized for maximum filter bypass

Risk: If triggered (even accidentally via keyword matching in a Telegram message), this skill could autonomously modify the agent's configuration to bypass safety filters on all future API calls.

### obliteratus - MODERATE RISK

A legitimate mechanistic interpretability tool for removing safety guardrails from open-weight model weights. Lower risk in this deployment because:
- Only affects locally-hosted models (not API-served models via LiteLLM/Azure)
- The .37 server uses Azure-hosted grok, not local inference for the agent
- Requires explicit GPU access and model download

## Fix Applied

### 1. Disabled Skills in Config

Added `skills.disabled` to `~/.hermes/config.yaml` on .37:

```yaml
skills:
  disabled:
  - godmode
  - obliteratus
```

This prevents the skill descriptions from being included in the `<available_skills>` block in the system prompt. The filtering is implemented in `agent/prompt_builder.py:build_skills_system_prompt()` which checks against the disabled list before adding each skill to the prompt.

### 2. Deleted Skill Directories

Removed the skill files from .37 to prevent execution even if somehow invoked:

```bash
rm -rf /home/sunny/project/hermes-agent/skills/red-teaming/godmode
rm -rf /home/sunny/project/hermes-agent/skills/mlops/inference/obliteratus
rm -rf /home/sunny/.hermes/skills/red-teaming/godmode
rm -rf /home/sunny/.hermes/skills/mlops/inference/obliteratus
```

Note: These directories may re-sync from other machines via Syncthing (the `~/project` folder is synced across all homelab servers). If they reappear, the `skills.disabled` config will still prevent them from being included in the system prompt, but the scripts would be back on disk.

### 3. Cleared Skills Prompt Cache

Deleted the cached skills prompt snapshot so the new config takes effect immediately:

```bash
rm -f /home/sunny/.hermes/.skills_prompt_snapshot.json
```

### 4. Restarted Gateway

```bash
systemctl --user restart hermes-gateway.service
```

## Verification

### Before Fix
```bash
# Replay original Hermes payload (with godmode/obliteratus in system prompt)
curl -X POST http://192.168.150.100:4004/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d @original_payload.json
# Result: HTTP 400 content_filter error
```

### After Fix
```bash
# Replay cleaned payload (without godmode/obliteratus)
curl -X POST http://192.168.150.100:4004/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d @cleaned_payload.json
# Result: HTTP 200 OK, model responds normally
```

### Confirmed No Prior Exploitation

Checked for artifacts indicating the jailbreak tools were previously used:
- `~/.hermes/prefill.json` — **Not found** (no jailbreak prefill was ever created)
- `agent.system_prompt` in config.yaml — **Not set** (no jailbreak system prompt override)
- No godmode/obliteratus output artifacts found

## Ongoing Considerations

### Syncthing Re-sync

The `~/project` folder syncs across all homelab machines. The deleted skill directories will reappear on .37 after the next Syncthing sync cycle. The `skills.disabled` config protects against prompt injection, but the scripts will be back on disk. Options:

- **Accept it**: `skills.disabled` is sufficient protection for the content filter issue
- **Add `.stignore` rule**: Create `~/project/hermes-agent/skills/red-teaming/.stignore` with `godmode` to prevent sync
- **Delete from all machines**: Remove from the upstream repo checkout on all servers

### Future Skill Additions

When pulling upstream updates from NousResearch/hermes-agent, new skills may be added that contain similar trigger language. After each `git pull`, check:

```bash
grep -ri "jailbreak\|bypass.*safety\|remove.*refusal\|uncensor" skills/*/SKILL.md skills/*/*/SKILL.md
```

If new triggering skills appear, add them to `skills.disabled` before restarting the gateway.

### Azure Content Filter Customization

If you need to run skills that trigger the Jailbreak Prompt Shield in the future, you can create a custom content filter in Azure AI Foundry:

1. Go to Azure AI Foundry portal > your resource > Guardrails + controls > Content filters
2. Create a new filter with Prompt Shields (Jailbreak) disabled
3. Assign it to the grok deployment

This is **not recommended** for production use — the default content filter provides valuable protection against prompt injection attacks.

### LiteLLM Error Code Mapping

Azure sometimes returns non-standard HTTP status codes through the LiteLLM proxy:
- **HTTP 431** ("Request Header Fields Too Large") — actually a content filter rejection, not a header size issue
- **HTTP 400** with `code: content_filter` — the correct/explicit content filter error

Both indicate the same underlying issue. LiteLLM correctly marks both as non-retryable.

## Files Changed

| File | Server | Change |
|------|--------|--------|
| `~/.hermes/config.yaml` | .37 | Added `skills.disabled: [godmode, obliteratus]` |
| `~/.hermes/.skills_prompt_snapshot.json` | .37 | Deleted (cache cleared) |
| `skills/red-teaming/godmode/` | .37 | Deleted directory |
| `skills/mlops/inference/obliteratus/` | .37 | Deleted directory |
| `~/.hermes/skills/red-teaming/godmode/` | .37 | Deleted directory (cached copy) |
| `~/.hermes/skills/mlops/inference/obliteratus/` | .37 | Deleted directory (cached copy) |

## Request Dump Files (Evidence)

Located at `~/.hermes/sessions/` on .37:
- `request_dump_20260401_151823_*.json` — First observed failure (Apr 1)
- `request_dump_cron_94a77561cc8f_20260402_*.json` — Cron job failure (Apr 2)
- `request_dump_cron_7d28fb02dfb8_*.json` — Additional cron failures

These files contain the full HTTP request body including the system prompt, and can be used to reproduce the issue for testing.
