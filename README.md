# Docker Web

My docker compose project for:

## Quick Start

Build the local Zola image (the Dockerfile downloads the Zola binary at build time):

```bash
docker compose build zola
```

Generate the static site:

```bash
docker compose run --rm zola build
```

## Serve locally (development)

Start the dev server (container listens on all interfaces). Open `http://localhost:1111`

```bash
docker compose run --rm --service-ports zola serve --interface 0.0.0.0
```

To force a specific base URL when serving locally (useful if assets reference production domain):

```bash
docker compose run --rm --service-ports zola serve --interface 0.0.0.0 --base-url http://localhost:1111
```

## Notes

- `compose.yml`: exposes port `1111:1111` for the `zola` service so the dev server is reachable on the host.
- `zola/Dockerfile`: builds a local Zola image by downloading the release binary.
