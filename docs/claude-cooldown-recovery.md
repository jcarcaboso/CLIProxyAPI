# Claude cooldown recovery

Claude Code's custom-endpoint API error frames keep the human error message but
drop extension fields and response headers. Returning `Retry-After` alone does
not give a downstream T3 Code thread a reset event.

For a 429 with a known future retry deadline, the Claude error envelope now ends
its message with:

```text
[CLIProxyAPI retry_at=1791141000]
```

The integer is an absolute Unix timestamp in seconds, rounded upward. It is
fixed when the upstream rejection is classified or credential selection fails,
not recalculated when the client response is written. For upstream Claude
rejections it uses the existing conservative reset parser, including its grace
period. For an exhausted credential pool it uses the proxy's eligibility time.
Known Codex quota-reset data and explicit OpenAI-compatible Retry-After signals
also retain their fixed deadlines when the request arrives through the Claude
endpoint. A heuristic per-minute fallback is not presented as a reported reset.
The deadline means a retry is eligible, not that an account is guaranteed to
have available quota.

The marker appears in both JSON error responses and terminal SSE errors. Only
typed internal errors with a known deadline can create it. Unknown reset times,
expired cooldowns, non-429 errors, and fast-mode credit-entitlement refusals do
not acquire a guessed reset timestamp. Existing error types and Retry-After
headers remain unchanged.

The matching managed Claude launcher in `sk1z0-group/homelab/nodes` recognizes
the marker only on a root synthetic API-rate-limit error. It emits a
`rate_limit_event` for `proxy_cooldown` before passing the original error and
result through. Unmodified T3 Code can then persist the reset and use its existing
`autoResumeLimitedThreads` and `snoozeLimitedThreads` preferences.

Both this proxy release and the managed launcher must be installed. Deploying
only one side does not fix proxy-backed automatic resume. Native Claude usage
events remain unchanged. No management API key or polling of private account
files is needed by the launcher.
