# Troubleshooting Timeline

## Initial Failure

Gemini for macOS launched Chrome and started a Google OAuth flow. The browser side completed successfully and called back to a local `localhost` server.

Gemini still failed to log in.

The diagnostic log showed the real failure happened after the callback:

```text
Exchanging authorization code
curl code 28 -> transport 86
Failed to exchange auth code
DEADLINE_EXCEEDED
```

## Network Split

Two tests revealed the split:

```bash
curl -x http://127.0.0.1:7897 https://oauth2.googleapis.com/token
```

This returned quickly.

```bash
curl --noproxy '*' https://oauth2.googleapis.com/token
```

This timed out.

That meant the proxy itself could reach Google, but Gemini's token exchange was not using the normal proxy path.

## Failed Direction

Environment variables and macOS system proxy were not enough:

```text
HTTP_PROXY
HTTPS_PROXY
ALL_PROXY
```

Gemini still timed out.

TUN mode looked like the right fix, but Clash Verge Rev failed to create the TUN interface:

```text
Start TUN listening error: configure tun interface: Connect: operation not permitted
```

The core was running in sidecar mode, so it did not have the privileges required for TUN.

## Actual Fix

Clash Verge Rev was upgraded to `2.5.1`.

After launch, logs showed:

```text
Starting core in service mode
服务成功启动核心
```

The core process was started by the privileged service, and TUN could finally create an interface:

```text
utun4
198.18.0.1/30
```

After enabling TUN, the no-proxy OAuth endpoint test returned quickly:

```text
http_code=404 time_total=1.213699
```

`404` was expected for the probe. The important change was that it no longer timed out.

Gemini login succeeded after retrying the OAuth flow.

## 2026-08 Recurrence

The same Gemini symptoms returned: a token-refresh loop with `transport error: 86` and `DEADLINE_EXCEEDED`.

This time the profile config already had a valid `tun:` block, and direct connectivity failed again:

```bash
curl --noproxy '*' https://oauth2.googleapis.com/token   # timed out
```

The Clash Verge log explained why TUN was not running:

```text
[Service] 服务需要重装，执行重装流程
[Service] failed to install service code: 1, details: 用户已取消 (-128)
[Core] Starting core in sidecar mode
```

The app had detected a service version mismatch and auto-prompted for an admin password reinstall on launch. The dialog was cancelled, so the app silently fell back to sidecar mode. With no root privileges, TUN could not run, and the settings page showed no usable Service Mode switch.

Enabling TUN in that state broke networking until it was switched back off.

## Recovery

Clash Verge was quit and reopened. This time the admin password dialog was accepted:

```text
[Service] install service
[Core] Starting core in service mode
[Service] 服务成功启动核心
```

The core then ran as root under the service process:

```text
50532  50529  root  verge-mihomo
```

TUN came up on `utun4` (`198.18.0.1`), and the probe returned quickly:

```text
http_code=404 time_total=1.083394 remote_ip=198.18.0.9
```

Gemini was quit and reopened. The existing session refreshed without a browser re-login:

```text
Request ...: 200 ... transport: 1
Received access token for 'user1'
signinStatus=signedIn
```
