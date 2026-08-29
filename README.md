# Fixing Gemini for macOS OAuth Login Timeout Behind Clash Verge Rev

This is a real-world troubleshooting note for a Gemini desktop login failure on macOS.

The browser OAuth flow looked successful, but Gemini for macOS stayed logged out. The root cause was not the browser callback. It was Gemini's internal `libcurl` request timing out while exchanging the OAuth authorization code for a token at `oauth2.googleapis.com`.

## Symptom

Gemini opens Chrome for Google login. Chrome completes the flow and calls back to the local `localhost` OAuth server, but the Gemini desktop app still fails to log in.

Gemini logs show the important part:

```text
Exchanging authorization code
curl code 28 -> transport 86
Failed to exchange auth code. Transport error: 86
OAuth flow failed: DEADLINE_EXCEEDED
```

On macOS, the logs are under:

```bash
~/Library/Caches/com.google.GeminiMacOS/Logs/
```

## What Was Ruled Out

- Browser callback to `localhost` worked.
- `HTTP_PROXY`, `HTTPS_PROXY`, and `ALL_PROXY` were set, but Gemini's internal OAuth exchange did not use them.
- macOS system proxy was configured, but Gemini still did not route the token request through it.
- Disabling IPv6 did not fix it.
- Direct `curl --noproxy '*' https://oauth2.googleapis.com/token` timed out before the fix.
- Explicit proxy access, such as `curl -x http://127.0.0.1:7897 ...`, worked.

## Root Cause

Gemini for macOS performed the OAuth token exchange with a network path that did not honor the normal proxy settings.

Because direct access to `oauth2.googleapis.com` was blocked or unreliable from the local network, Gemini's token exchange timed out.

The reliable fix was to make Clash Verge Rev's TUN mode work correctly so Gemini's direct traffic would be captured at the network layer.

In this case, Clash Verge Rev was running the core in sidecar mode, and TUN failed with:

```text
Start TUN listening error: configure tun interface: Connect: operation not permitted
```

After upgrading Clash Verge Rev and getting the core to run in service mode, TUN could create a `utun` interface and route Gemini's token exchange successfully.

## Fix Summary

1. Upgrade Clash Verge Rev to a version where service mode works correctly.
2. Confirm the core starts in service mode, not sidecar mode.
3. Enable TUN mode.
4. Verify a `utun` interface is created.
5. Verify direct, no-proxy connectivity to Google's OAuth token endpoint no longer times out.
6. Retry Gemini login.

## Verification Commands

Check Clash Verge Rev version:

```bash
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' \
  '/Applications/Clash Verge.app/Contents/Info.plist'
```

Check whether the core is launched by the privileged service:

```bash
ps -axo pid,ppid,user,comm,args | rg -i 'clash-verge|verge-mihomo|mihomo'
```

Good sign: `verge-mihomo` is started by the root service process.

Check Clash Verge app logs:

```bash
tail -n 120 \
  "$HOME/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/logs/latest.log"
```

Good signs:

```text
Starting core in service mode
Service started core successfully
```

Check TUN state through the mihomo control socket:

```bash
curl --unix-socket /tmp/verge/verge-mihomo.sock -sS http://unix/configs \
  | python3 -m json.tool \
  | sed -n '1,45p'
```

Good sign:

```json
"tun": {
  "enable": true,
  "device": "utun4"
}
```

Check the macOS interface:

```bash
ifconfig | rg -n 'utun|198\.18|flags|inet '
```

Check OAuth endpoint connectivity without explicit proxy:

```bash
curl --noproxy '*' --connect-timeout 10 -sS -o /dev/null \
  -w 'http_code=%{http_code} time_connect=%{time_connect} time_total=%{time_total} remote_ip=%{remote_ip}\n' \
  https://oauth2.googleapis.com/token
```

For this GET-style probe, `404` is fine. The key is that it returns quickly instead of timing out.

Example good result:

```text
http_code=404 time_connect=0.002271 time_total=1.213699 remote_ip=173.194.174.95
```

## Notes

This was not primarily a VPS problem, although an unstable VPS node can still make the experience worse. The decisive failure was that Gemini's token exchange was not going through the normal proxy path, while Clash Verge Rev's TUN mode was not actually working.

Once Clash Verge Rev ran in service mode and TUN was enabled, Gemini login succeeded.

## 2026-08 Recurrence: A Stale Service Silently Falls Back to Sidecar Mode

Months later, the same Gemini symptoms returned: the app launched but stayed stuck in a token-refresh loop:

```text
Failed to refresh access token. Transport error: 86
HandleAuthStatus: status=DEADLINE_EXCEEDED (TIMEOUT_EXCEEDED)
```

This time the profile config already contained a valid `tun:` block, so the config was not the problem. The failure was one layer lower.

### What Actually Happened

The Clash Verge Rev app had been updated, but the privileged service under `/Library/PrivilegedHelperTools` no longer matched it. On startup, the app detects the version mismatch and auto-prompts to reinstall the service with an admin password dialog:

```text
[Service] 服务需要重装，执行重装流程
[Service] install service
[Service] failed to install service code: 1, details: 用户已取消 (-128)
[Core] Starting core in sidecar mode
```

The password dialog was dismissed, and the app **silently fell back to sidecar mode**. There is no persistent error anywhere in the UI. Worse, while the service is in this stale state the settings page does not offer a usable "Service Mode" toggle at all, which makes the root cause hard to find.

Two extra traps appeared during recovery:

- Writing `tun: enable: true` into the profile config or patching the mihomo API directly did nothing. The runtime TUN state stayed `"enable": false` — the app's own toggle overwrites the config at runtime.
- Enabling TUN while the core was still in sidecar mode broke networking entirely until TUN was switched back off.

### The Fix

1. Quit Clash Verge completely and reopen it.
2. When the admin password dialog appears, **enter the password**. This reinstalls the service.
3. Confirm the logs show service mode:

   ```text
   [Core] Starting core in service mode
   [Service] 服务成功启动核心
   ```

4. Confirm the core is root-owned and parented to the service process:

   ```text
   50532  50529  root  verge-mihomo ...
   ```

5. Enable TUN, then verify a `utun` interface with a `198.18.x.x` address appears and the no-proxy probe returns quickly.
6. Quit and reopen Gemini. The existing session recovered without a browser re-login:

   ```text
   Request ...: 200 ... transport: 1
   Received access token for 'user1'
   signinStatus=signedIn
   ```

### Lessons

- After updating Clash Verge Rev, never dismiss the admin password prompt at startup — cancelling it downgrades to sidecar mode silently.
- The decisive log line is `Starting core in service mode` vs `Starting core in sidecar mode`. Check it first.
- Order matters: service mode first, TUN second. TUN on a sidecar core takes the network down.
- A `tun:` block inside the profile is not enough; the app-level TUN toggle governs the runtime state.

## Related Clues

- Gemini CLI reports from users with OAuth token exchange timeouts against `oauth2.googleapis.com/token`.
- Clash Verge Rev macOS reports where TUN fails with `operation not permitted`.
- Clash Verge Rev service/sidecar mode regressions where the core runs without the privileges needed for TUN.
