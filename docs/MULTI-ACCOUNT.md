# Multi-Account Setup

Add multiple Google accounts to increase your combined quota. The plugin automatically rotates between accounts when one is rate-limited.

```bash
opencode auth login  # Run again to add more accounts
```

---

## Account Management Menu

When running `opencode auth login` with existing accounts, an interactive TUI menu appears:

```
┌  Manage accounts
│
◆  Select account

│  ● Add new account
│  ○ user1@gmail.com [active]  used today
│  ○ user2@gmail.com [rate-limited]  used yesterday
│  ○ Delete all accounts
│
│  ↑/↓ to select • Enter: confirm
└
```

Use arrow keys to navigate, Enter to select. The menu supports:
- **Keyboard navigation** — Up/Down arrows to move between options
- **Color-coded status** — Green for active, yellow for rate-limited, red for expired
- **Relative timestamps** — "today", "yesterday", "3d ago", etc.

### Menu Options

| Option | Description |
|--------|-------------|
| **Add new account** | Add another Google account to the pool |
| **\<account email\>** | Select to view account details and management options |
| **Delete all accounts** | Remove all accounts and start fresh (requires confirmation) |

### Account Status Badges

| Badge | Color | Meaning |
|-------|-------|---------|
| `[active]` | Green | Account is available for use |
| `[rate-limited]` | Yellow | Account has hit quota limits (will auto-recover) |
| `[expired]` | Red | Token needs refresh |

### Account Details View

Selecting an account shows detailed information and actions:

```
Account: user1@gmail.com [active]
Added: 1/15/2026
Last used: today

┌  Account options
│
◆  Select action

│  ● Back
│  ○ Refresh token
│  ○ Delete this account
│
│  ↑/↓ to select • Enter: confirm
└
```

| Action | Color | Description |
|--------|-------|-------------|
| **Back** | — | Return to main menu |
| **Refresh token** | Cyan | Re-authenticate this account (fixes expired tokens) |
| **Delete this account** | Red | Remove only this account from the pool (requires confirmation) |

> **Note:** Destructive actions (delete, refresh) require confirmation before executing.

---

## Load Balancing Behavior

- **Sticky account selection** — Sticks to the same account until rate-limited (preserves Anthropic's prompt cache)
- **Per-model-family limits** — Rate limits tracked separately for Claude and Gemini models
- **Dual quota pools for Gemini** — Automatic fallback between Antigravity quota and Gemini CLI quota before switching accounts
- **Smart retry threshold** — Short rate limits (≤5s) are retried on same account
- **Exponential backoff** — Increasing delays for consecutive rate limits

---

## Dual Quota Pools

For Gemini models, the plugin accesses **two independent quota pools** per account:

| Quota Pool | When Used |
|------------|-----------|
| **Antigravity** | Primary (tried first) |
| **Gemini CLI** | Fallback when Antigravity is rate-limited |

This effectively **doubles your Gemini quota** per account.

To enable automatic fallback between pools, set in `antigravity.json`:

```json
{
  "quota_fallback": true
}
```

---

## Adding Accounts

Select **Add new account** from the management menu to add more accounts while keeping existing ones.

For non-interactive terminals (CI/CD, scripts), the plugin falls back to a simple text prompt:

```
2 account(s) saved:
  1. user1@gmail.com
  2. user2@gmail.com

(a)dd new account(s) or (f)resh start? [a/f]:
```

---

## Account Storage

Accounts are stored in `~/.config/opencode/antigravity-accounts.json`:

```json
{
  "version": 3,
  "accounts": [
    {
      "email": "user1@gmail.com",
      "refreshToken": "1//0abc...",
      "projectId": "my-gcp-project",
      "addedAt": 1737100800000,
      "lastUsed": 1737187200000
    },
    {
      "email": "user2@gmail.com",
      "refreshToken": "1//0xyz...",
      "addedAt": 1737014400000,
      "lastUsed": 1737100800000
    }
  ],
  "activeIndex": 0,
  "activeIndexByFamily": {
    "claude": 0,
    "gemini": 0
  }
}
```

> ⚠️ **Security:** This file contains OAuth refresh tokens. Treat it like a password file.

### Fields

| Field | Description |
|-------|-------------|
| `email` | Google account email |
| `refreshToken` | OAuth refresh token (auto-managed) |
| `projectId` | Optional. Required for Gemini CLI models. See [Troubleshooting](TROUBLESHOOTING.md#gemini-cli-permission-error). |
| `addedAt` | Timestamp when account was added |
| `lastUsed` | Timestamp of last API request with this account |
| `activeIndex` | Currently active account index |
| `activeIndexByFamily` | Per-model-family active account (claude/gemini tracked separately) |

---

## Token Revocation

If Google revokes a token (e.g., password change, security event), you'll see `invalid_grant` errors. The plugin automatically removes invalid accounts.

**To refresh a single account:**
1. Run `opencode auth login`
2. Select the affected account from the menu
3. Choose **Refresh token**

**To manually reset all accounts:**

```bash
rm ~/.config/opencode/antigravity-accounts.json
opencode auth login
```

---

## Parallel Sessions (oh-my-opencode)

When using oh-my-opencode with parallel subagents, multiple processes may select the same account, causing rate limit errors.

**Solution:** Enable PID-based offset in `antigravity.json`:

```json
{
  "pid_offset_enabled": true
}
```

This distributes sessions across accounts based on process ID.

Alternatively, add more accounts via `opencode auth login`.

---

## Account Selection Strategies

Configure in `antigravity.json`:

```json
{
  "account_selection_strategy": "hybrid"
}
```

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `sticky` | Same account until rate-limited | Prompt cache preservation |
| `round-robin` | Rotate to next account on every request | Maximum throughput |
| `hybrid` | Deterministic selection based on health score + token bucket + LRU | Best overall distribution |

See [Configuration](CONFIGURATION.md#account-selection) for more details.
