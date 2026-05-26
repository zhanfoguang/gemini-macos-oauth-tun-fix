# Tweet Thread Draft

1. Debugged a weird Gemini for macOS login failure today.

Chrome OAuth said login succeeded, but the desktop app stayed logged out.

The important clue was buried in Gemini's local logs:

`curl code 28 -> transport 86 -> DEADLINE_EXCEEDED`

2. The browser callback was fine.

Gemini received the OAuth code from `localhost`.

The failure happened after that, when Gemini's internal `libcurl` tried to exchange the code for a token at:

`oauth2.googleapis.com/token`

3. Proxy env vars did not help.

`HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and macOS system proxy were all set, but Gemini still timed out.

Explicit proxy worked:

`curl -x 127.0.0.1:7897 ...`

Direct access timed out:

`curl --noproxy '*' ...`

4. The fix was not "launch it from Terminal".

The fix was making Clash Verge Rev TUN actually work, because Gemini was not honoring the ordinary proxy path.

TUN had been failing with:

`operation not permitted`

5. Root cause on the proxy side:

Clash Verge Rev was running mihomo in sidecar mode, not service mode.

That meant no privilege to create the TUN interface.

After upgrading Clash Verge Rev and getting service mode working, TUN created `utun4`.

6. Verification after the fix:

`curl --noproxy '*' https://oauth2.googleapis.com/token`

changed from timeout to a fast response.

The response was `404`, which is fine for a GET probe. The point is: no timeout.

7. After that, Gemini login worked immediately.

Lesson: when desktop OAuth succeeds in browser but app login fails, check the post-callback token exchange path.

Some apps do not respect your proxy env/system proxy the way you think they do.

8. Wrote up the full runbook here:

https://github.com/zhanfoguang/gemini-macos-oauth-tun-fix
