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
