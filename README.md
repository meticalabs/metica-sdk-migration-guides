# MaxSdk → MeticaSdk Migration Guide for AI Agents

Use these instruction files to have an AI coding agent migrate your Unity project from **AppLovin MaxSdk 8.6.0** to **MeticaSdk 2.2.2**.

## Choose Your Scenario

| Scenario | File | When to Use |
|----------|------|-------------|
| **Complete Replacement** | [`migrate-complete-replacement.md`](migrate-complete-replacement.md) | You want to fully migrate all MaxSdk calls to MeticaSdk in one pass. Best for most projects. |
| **A/B Testing** | [`migrate-ab-testing.md`](migrate-ab-testing.md) | You want to run both SDKs side-by-side to validate MeticaSdk performance (revenue, fill rates) against your MaxSdk baseline before committing. |

## How to Use With Your AI Tool

### Claude Code (VS Code or CLI)
```
# Option 1: Reference as file context
@migrate-complete-replacement.md Migrate my project from MaxSdk to MeticaSdk

# Option 2: Copy into your project's CLAUDE.md for auto-loading
cp migrate-complete-replacement.md /path/to/your-unity-project/CLAUDE.md
```

### GitHub Copilot (VS Code)
```
# Option 1: Attach file in Copilot chat
# Click the paperclip icon → select the migration file → type your prompt

# Option 2: Place as project instructions (auto-loaded for all chats)
mkdir -p /path/to/your-unity-project/.github
cp migrate-complete-replacement.md /path/to/your-unity-project/.github/copilot-instructions.md
```

### Cursor
```
# Option 1: Attach file in chat with @file
@migrate-complete-replacement.md Migrate my project from MaxSdk to MeticaSdk

# Option 2: Place as project rules (auto-loaded)
cp migrate-complete-replacement.md /path/to/your-unity-project/.cursorrules
```

### Any Other AI (ChatGPT, Gemini, etc.)
Copy-paste the contents of the migration file into the chat, then ask the AI to migrate your code.

## Prerequisites

Before starting any migration scenario, ensure:

1. **AppLovin MAX Unity Plugin v8.2.0 or later** — MeticaSdk requires this minimum version. Verify by checking `MaxSdk.Version` in code or the plugin's `CHANGELOG.md` / `package.json` in your project. If below 8.2.0, update the MAX plugin first.
2. **MeticaSdk 2.2.2** `.unitypackage` imported into the project
3. **Credentials ready:**
   - Metica API Key (from Metica platform)
   - Metica App ID (from Metica platform)
   - AppLovin MAX SDK Key (your existing key — reused by MeticaSdk)
4. **Ad Unit IDs** — keep your existing IDs, they work unchanged with MeticaSdk

## Reference

- [API Comparison Document](../MaxSdk-8.6.0-vs-MeticaSdk-2.2.2-API-Comparison.md) — full side-by-side API comparison
- [MeticaSdk Official Docs](https://docs.metica.com/api/unity-sdk/unity-sdk-2) — official integration guide
