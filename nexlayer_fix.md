# Nexlayer deploy fix guidance (pinned)

PIN the existing Dockerfile and nexlayer.yaml on this branch. Do NOT regenerate
the Dockerfile. This app is built from source so a source patch
(src/backend/server.ts + src/backend/core/http/middleware/hostRedirects.ts) is
compiled into the image.

Root cause fixed: puter used Express's default `subdomain offset` of 2, so the
4-label deploy host `relaxed-weasel-puter.cloud.nexlayer.ai` was misread as a
user-site subdomain and `/` 302-redirected to the certless
`site.relaxed-weasel-puter.cloud.nexlayer.ai` (browser ERR_SSL_PROTOCOL_ERROR).
The fix derives the subdomain offset from `config.domain` and serves the GUI on
the apex with no cross-host redirect. Keep the Dockerfile multi-stage source
build and the boot command in nexlayer.yaml as-is.
