# LinkHarbor for Terraria (TShock)

A single-file Python agent that polls a Terraria server running
[TShock](https://github.com/Pryaxis/TShock) and pushes the live state to your
[LinkHarbor](https://linkharbor.com) bio page: a game-server card with
Online/Offline, "3 / 8" players, player names, the world name and the server
version, plus join/leave events for push notifications to your followers.

It uses TShock's REST API (`/v2/server/status`, default port 7878), so
nothing is installed on the server beyond a REST token. Standard library only,
Python 3.10 or newer. Vanilla Terraria servers without TShock have no REST API and are
not supported.

Part of the [LinkHarbor integrations](https://github.com/linkharbor/integrations)
family, one repo per game or tool.

## Setup

```bash
git clone https://github.com/linkharbor/integration-terraria
cd integration-terraria

export LH_INGEST_URL="https://dock.at-harbor.com/v1/ingest/123"   # dashboard: Integrations, Custom (API)
export LH_INGEST_SECRET="…"
export TSHOCK_TOKEN="…"                                            # see "Getting a REST token"

python3 agent.py --host 203.0.113.10 --port 7878 \
  --address "203.0.113.10:7777" --url "https://discord.gg/xyz" --interval 60
```

Start with `--once -v`. Run it as a service with `integration-terraria.service`
(systemd; secrets go into `/etc/integration-terraria.env`, `chmod 600`). If the
server runs on the same machine, `--host 127.0.0.1` works even when the REST
port is firewalled from the outside.

## Events sent

| Event | When | Template variables |
|---|---|---|
| `server.online` | the first successful poll, or the server is back after being unreachable | `{name}`, `{players_online}`, `{players_max}`, `{map}`, `{version}` |
| `server.offline` | the server stopped answering, or rejected the token | `{name}` |
| `player.joined` | a new nickname appears in the player list | `{player}` and the above |
| `player.left` | a nickname disappears | `{player}` and the above |

The state is sent with every event and every `--heartbeat` minutes (default
10). `{map}` is the world name. `--no-player-names` sends counts only.

Suggested push rule (dashboard, Integrations, Push rules, Custom (API), event `player.joined`): batch window 10 min, title "🎮 {count} players joined", body
"{player_list} are now on {name}", minimum pause 5 min.

## Options

```
--host HOST            server IP or hostname (required)
--port PORT            TShock REST port (default 7878)
--token TOKEN          TShock REST token (or env TSHOCK_TOKEN)
--name NAME            card title (default: the server's name)
--address ADDRESS      address shown on the card, e.g. 203.0.113.10:7777
--url URL              link behind the card's button
--cta TEXT             button label (default "Join server")
--game TEXT            game label (default "Terraria")
--image-url URL        banner image (large layout)
--interval SECONDS     polling interval (default 60, minimum 5)
--heartbeat MINUTES    state-only update interval (default 10, 0 = only on change)
--no-player-names      send counts only
--once                 poll once, send, exit
-v                     verbose logging
```

## Getting a REST token

TShock's REST API is off by default. In `tshock/config.json` (next to the
server) set:

```json
"RestApiEnabled": true,
"RestApiPort": 7878,
"ApplicationRestTokens": {
  "a-long-random-string": { "Username": "linkharbor", "UserGroupName": "default" }
}
```

The key is the token you pass as `TSHOCK_TOKEN`; `Username` is the TShock
account the token acts as (create one with `/user add <name> <password> <group>`
in the server console). Restart the server after editing the file. The status endpoint
only needs a valid token, not admin rights. The token is sent as a query
parameter over plain HTTP, so keep the REST port on a private network or
behind a firewall.

## License

[MIT](LICENSE). © 2026 EMI Solutions OÜ.
