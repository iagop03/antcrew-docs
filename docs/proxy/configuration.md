# Configuration

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `PLATFORM_URL` | yes | URL of antcrew-platform API |
| `PLATFORM_INTERNAL_KEY` | yes | Service-to-service key for key lookup |
| `PORT` | no | Port to listen on (default: 8080) |
| `LOG_FORMAT` | no | `json` or `text` (default: `json`) |

## Docker

```yaml
services:
  proxy:
    image: ghcr.io/iagop03/antcrew-proxy:latest
    ports:
      - "8080:8080"
    environment:
      PLATFORM_URL: https://platform-int.antcrew.org
      PLATFORM_INTERNAL_KEY: ${INTERNAL_KEY}
```

## Supported ports (Cloudflare proxy-compatible)

The proxy listens on port 8080 by default, which is in Cloudflare's list of supported HTTP ports, making it compatible with Cloudflare proxy without any additional configuration.
