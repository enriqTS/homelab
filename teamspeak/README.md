# TeamSpeak 6 - WebQuery

TeamSpeak is deployed on the Mini PC by `ansible-playbook playbooks/minipc.yml`.
On its first run, Ansible creates random database credentials in
`/opt/homelab/teamspeak/.env`; this file is mode `0600`, is not committed, and
must be backed up with the named Docker volumes. Connect TeamSpeak clients over
Tailscale at `henrique-notebook.tail640e58.ts.net:9987`.

WebQuery is the JSON-based REST API built into the TeamSpeak 6 server. It lets you
read server state and run admin commands over plain HTTP — no extra tools needed.

## 1. Enable it

Already enabled in `docker-compose.yml`:

```yaml
ports:
  - "10080:10080/tcp" # WebQuery port
environment:
  - TSSERVER_QUERY_HTTP_ENABLED=true
```

Recreate the container to apply:

```bash
docker compose up -d
```

Test that it answers:

```bash
curl -i http://localhost:10080/
# Should return 401 (no API key yet) — that's expected.
```

## 2. Create an API key

WebQuery authenticates with an **API key** (`x-api-key` header), not the serveradmin password.

### Option A — from the TeamSpeak 6 client

1. Connect to your server with admin rights.
2. `Server Settings → API Keys → Generate Key`
3. Pick a scope:
   - `manage` — full admin access
   - `write`  — most write commands
   - `read`   — read-only
4. Copy the key. It is shown only once.

### Option B — from the command line

If SSHQuery is enabled (`TSSERVER_QUERY_SSH_ENABLED=true`, port 10022):

```bash
ssh serveradmin@localhost -p 10022
apikeyadd scope=manage lifetime=180
```

Store the key in a safe place (e.g. a password manager). If you lose it, delete and
recreate it.

## 3. Basic requests

Every request needs the `x-api-key` header. The URL shape is:

```
http://localhost:10080/<virtualserver_id>/<command>?<params>
```

Global commands (no server ID): `whoami`, `serverlist`, `instanceinfo`, `hostinfo`.

```bash
KEY="YOUR_API_KEY"

# Who am I / is my key valid?
curl -s -H "x-api-key: $KEY" http://localhost:10080/whoami

# List all virtual servers
curl -s -H "x-api-key: $KEY" http://localhost:10080/serverlist

# List clients on virtual server 1 (pretty-printed)
curl -s -H "x-api-key: $KEY" http://localhost:10080/1/clientlist | json_pp

# Server info for virtual server 1
curl -s -H "x-api-key: $KEY" http://localhost:10080/1/serverinfo | json_pp

# Send a global message
curl -s -H "x-api-key: $KEY" -X POST \
  -d '{"msg":"hello from curl","msg_targetmode":3}' \
  http://localhost:10080/1/gm
```

## 4. Response format

Every response has the same shape:

```json
{
  "status": { "code": 0, "message": "ok" },
  "body": [ ... ]
}
```

`status.code == 0` means success; anything else is an error and `status.message`
explains it.

## 5. Notes

- The API is unencrypted HTTP by default. Only expose port 10080 on trusted
  networks, or put it behind your reverse proxy / Tailscale.
- Available endpoints map to the ServerQuery command names
  (`clientlist`, `serverinfo`, `channelcreate`, `banadd`, ...). Full reference:
  the OpenAPI docs the server exposes at `http://localhost:10080/swagger-ui` or
  the official repo: https://github.com/teamspeak/teamspeak6-server
- Want a browser GUI instead of curl? See the `ts6-manager` project
  (https://github.com/clusterzx/ts6-manager) — evaluate it before trusting it.
