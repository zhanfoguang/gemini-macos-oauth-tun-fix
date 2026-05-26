# Runbook

Use this when Gemini for macOS opens Chrome, Chrome says login succeeded, but the desktop app remains logged out.

## 1. Inspect Gemini Logs

```bash
ls -t "$HOME/Library/Caches/com.google.GeminiMacOS/Logs"/diagnostic*.log | head
tail -n 120 "$(ls -t "$HOME/Library/Caches/com.google.GeminiMacOS/Logs"/diagnostic*.log | head -1)"
```

Look for:

```text
Exchanging authorization code
curl code 28
transport: 86
DEADLINE_EXCEEDED
```

## 2. Compare Direct vs Proxy Connectivity

Direct path:

```bash
curl --noproxy '*' --connect-timeout 8 -sS -o /dev/null \
  -w 'http_code=%{http_code} time_total=%{time_total}\n' \
  https://oauth2.googleapis.com/token
```

Explicit proxy path:

```bash
curl -x http://127.0.0.1:7897 --connect-timeout 8 -sS -o /dev/null \
  -w 'http_code=%{http_code} time_total=%{time_total}\n' \
  https://oauth2.googleapis.com/token
```

If explicit proxy works but direct times out, Gemini may fail unless TUN captures the traffic.

## 3. Check Clash Verge Rev Service Mode

```bash
ps -axo pid,ppid,user,comm,args | rg -i 'clash-verge|verge-mihomo|mihomo'
```

The desired shape is:

```text
root ... clash-verge-service
root ... verge-mihomo ...
```

Then check:

```bash
tail -n 120 \
  "$HOME/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/logs/latest.log"
```

Look for:

```text
Starting core in service mode
服务成功启动核心
```

## 4. Check TUN

```bash
curl --unix-socket /tmp/verge/verge-mihomo.sock -sS http://unix/configs \
  | python3 -m json.tool \
  | sed -n '1,45p'
```

Expected:

```json
"tun": {
  "enable": true,
  "device": "utun..."
}
```

Also verify:

```bash
ifconfig | rg -n 'utun|198\.18|flags|inet '
```

## 5. Retry Gemini Login

Once direct no-proxy access to `oauth2.googleapis.com/token` returns quickly, reopen Gemini and retry login.

Do the Google account selection and OAuth consent manually. Treat that as a sensitive step.
