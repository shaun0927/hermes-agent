# Apply A + B + Codex MCP wiring on fly app `rivetta`

Run on your laptop. Steps in order — do not skip.

## 0. Login

```bash
flyctl auth login
```

## 1. Pre-flight — what's already there

Before changing anything, capture current state so we can roll back:

```bash
flyctl ssh console -a rivetta -C 'cat /opt/data/config.yaml' > /tmp/rivetta-config.yaml.bak
flyctl ssh console -a rivetta -C 'ls -la /opt/data/.codex 2>/dev/null || echo NO_CODEX_DIR'
flyctl ssh console -a rivetta -C 'which omx ouroboros codex'
```

Expected before patch: `omx` returns "not found", `ouroboros` and `codex` resolve.

## 2. Push the new image (Dockerfile already patched)

```bash
cd /Users/jh0927/Workspace/hermes-agent-base
git status                 # confirm Dockerfile diff includes the oh-my-codex line
flyctl deploy --remote-only -a rivetta
```

Wait for green deploy. Verify:

```bash
flyctl ssh console -a rivetta -C 'which omx && omx --version'
```

Should now resolve to `/usr/local/bin/omx` or similar.

## 3. Wire ouroboros into hermes (`/opt/data/config.yaml`)

Push the snippet onto the volume:

```bash
flyctl ssh sftp shell -a rivetta
# inside sftp:
put /Users/jh0927/Workspace/hermes-agent-base/deploy-configs/hermes-mcp-snippet.yaml /opt/data/hermes-mcp-snippet.yaml
quit
```

Merge it in (idempotent — re-applying is safe):

```bash
flyctl ssh console -a rivetta
# inside container:
if ! grep -q "^mcp_servers:" /opt/data/config.yaml; then
  cat /opt/data/hermes-mcp-snippet.yaml >> /opt/data/config.yaml
else
  echo "mcp_servers: already present — edit /opt/data/config.yaml manually to add the ouroboros entry"
fi
exit
```

## 4. Wire ouroboros into Codex CLI (`/opt/data/.codex/config.toml`)

Codex CLI ships an `add` subcommand that edits `config.toml` for you, so prefer that over hand-editing:

```bash
flyctl ssh console -a rivetta
# inside container:
mkdir -p /opt/data/.codex
codex mcp add ouroboros \
  --command ouroboros \
  -- mcp serve --transport stdio --llm-backend codex
codex mcp list                # verify
exit
```

If `codex mcp add` flag syntax differs in 0.128.0+, fall back to manual append:

```bash
flyctl ssh sftp shell -a rivetta
put /Users/jh0927/Workspace/hermes-agent-base/deploy-configs/codex-mcp-snippet.toml /opt/data/.codex/codex-mcp-snippet.toml
quit

flyctl ssh console -a rivetta
# inside container:
cat /opt/data/.codex/codex-mcp-snippet.toml >> /opt/data/.codex/config.toml
codex mcp list
exit
```

## 5. Codex authentication

omx + codex need an authenticated session. Two paths:

**Path A — copy your laptop's auth file (fastest if you already use codex locally):**

```bash
flyctl ssh sftp shell -a rivetta
put ~/.codex/auth.json /opt/data/.codex/auth.json
quit

flyctl ssh console -a rivetta -C 'chmod 600 /opt/data/.codex/auth.json && chown 10000:10000 /opt/data/.codex/auth.json'
```

**Path B — login from inside the container (device-code flow):**

```bash
flyctl ssh console -a rivetta
# inside container:
codex login        # follow the device-code URL in your laptop browser
exit
```

## 6. Restart hermes so it re-reads config and respawns MCP children

```bash
flyctl machine list -a rivetta                       # note machine id
flyctl machine restart <machine-id> -a rivetta
flyctl logs -a rivetta                               # tail; expect ouroboros mcp server startup line
```

## 7. End-to-end smoke test

```bash
flyctl ssh console -a rivetta
# inside container:
hermes status                                         # ouroboros should appear under MCP servers
ouroboros mcp doctor                                  # green
omx --status                                          # codex billing/provider sane
omx "eco: print hello and exit"                       # smallest possible omx run
exit
```

Then trigger from outside: send a message via whatever channel the gateway listens on
("ooo interview" or "use ouroboros to interview me about X"). hermes' LLM should see
the ouroboros_* tools in its tool list.

## Rollback

If anything breaks:

```bash
flyctl ssh sftp shell -a rivetta
put /tmp/rivetta-config.yaml.bak /opt/data/config.yaml
quit

flyctl machine restart <machine-id> -a rivetta
```

For the image, redeploy the previous git SHA:

```bash
cd /Users/jh0927/Workspace/hermes-agent-base
git stash                                             # park the Dockerfile change
flyctl deploy --remote-only -a rivetta
git stash pop                                         # bring it back when ready
```
