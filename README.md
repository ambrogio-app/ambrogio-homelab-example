# Ambrogio homelab example

A complete home installation of [Ambrogio](https://ambrogio-app.github.io/ambrogio-docs/), one Docker Compose
file per service, as a starting point for your own homelab:

| Folder | Runs | Where |
|---|---|---|
| `proxy/` | Caddy: `/api/*` to the backend, everything else to the web chat | homelab server |
| `ambrogio/` | backend, its PostgreSQL archive, the web chat and the Open-Meteo MCP server for the weather agent | homelab server |
| `homeassistant/` | Home Assistant on the host network | homelab server |
| `satellite/` | one voice satellite | a Raspberry Pi or another device in a room |

The agents in `ambrogio/agents/` use these services:

| Agent | Uses |
|---|---|
| `generic` | general knowledge only |
| `home-assistant` | the "Model Context Protocol Server" integration of Home Assistant |
| `memory` | the backend's own MCP server at `/mcp/memory`, to search past conversations |
| `weather` | the `open-meteo-mcp` container |
| `web-search` | the public MCP server of [Parallel](https://parallel.ai/) |

All of them use Mistral AI models; any provider of the [agent configuration](https://ambrogio-app.github.io/ambrogio-docs/reference/agent-configuration/) works.

> **This is an example, not a release.** It is not updated with every release of Ambrogio. It was written for
> backend `v0.17.0`, web chat `v0.3.1` and satellite `v0.8.1`; before using another release, read its notes in
> [Versions and compatibility](https://ambrogio-app.github.io/ambrogio-docs/reference/versions/) and adapt the
> files. The documentation is the reference for every field and variable.

## Start

On the homelab server, from the root of this repository:

1. Copy `.env.example` to `.env` and fill in every value.
2. In `ambrogio/web/config.json`, set `backendUrl` to `https://` followed by your `AMBROGIO_DOMAIN`.
3. In `ambrogio/agents/weather.json`, replace `HOME_TOWN` with the town used when nobody names a place.
4. Create the network the proxy shares with the services, then start the services in this order:

   ```bash
   docker network create homelab_proxy
   docker compose --env-file .env -f homeassistant/docker-compose.yml up -d
   docker compose --env-file .env -f ambrogio/docker-compose.yml up -d
   docker compose --env-file .env -f proxy/docker-compose.yml up -d
   ```

5. In Home Assistant, add the "Model Context Protocol Server" integration, expose to Assist the entities Ambrogio may
   use, create a long-lived access token and put it in `HOME_ASSISTANT_TOKEN`. Then restart the backend:
   `docker compose --env-file .env -f ambrogio/docker-compose.yml restart ambrogio`.

The backend connects to the MCP server of each agent once, right after it starts listening. An agent whose server
does not answer stays unavailable until the backend restarts, with an `agent_mcp_unavailable` line in its log: start
Home Assistant before the backend, or restart the backend once Home Assistant is up.

Point `AMBROGIO_DOMAIN` to the homelab server in your home DNS. Caddy issues the certificate from its own local
authority: install Caddy's root certificate, `/data/caddy/pki/authorities/local/root.crt` in the `caddy-data`
volume, on the family's devices, or replace `tls internal` in `proxy/Caddyfile` with a certificate for a public domain.

## Satellite

Copy `satellite/` to the device in the room, copy `satellite.env.example` to `satellite.env`, set
`AMBROGIO_BACKEND_URL` to port `3000` of the homelab server and give each satellite its own `AMBROGIO_CLIENT_ID`
and `AMBROGIO_CLIENT_NAME`. Then `docker compose up -d` on the device. Microphone, speaker and wake word are set with
the variables of the [satellite configuration](https://ambrogio-app.github.io/ambrogio-docs/reference/satellite-configuration/).

## What stays private

The backend has no accounts: anyone who reaches it can talk to it. Keep port `3000` and the domain inside the home
network, or behind a VPN. `/mcp/memory` is not forwarded by the proxy. The data live in Docker volumes: back up
`ambrogio-database` with `pg_dump` and `homeassistant-config` as Home Assistant explains. Never commit `.env` or
`satellite.env`.

## Checks

`scripts/check` validates every compose file with the variables of `.env.example`, parses the JSON files and loads
the agents with the backend release named in `.env.example`. CI runs it on every pull request, together with a scan
for secrets. Enable the pre-commit hook in each clone with `git config core.hooksPath .githooks`; it needs
`gitleaks` on `PATH`.

## License

MIT, like Ambrogio.
