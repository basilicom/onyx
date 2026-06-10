# Onyx v4 — Basilicom-Deployment (Portainer + Traefik)

Fork von [onyx-dot-app/onyx](https://github.com/onyx-dot-app/onyx) auf Tag
`v4.0.7` (OpenSearch-only, kein Vespa). Dieses Verzeichnis enthält die
Basilicom-spezifischen Deploy-Dateien — die Upstream-Compose bleibt unverändert.

## Dateien

| Datei | Zweck |
|---|---|
| `docker-compose.yml` | Upstream-Basis (unverändert) |
| `docker-compose.basilicom.yml` | **Override**: Traefik-Labels, externes Netz, keine Host-Ports |
| `env.basilicom.template` | Env-Vorlage (Secrets NICHT committen) |

## Deploy via Portainer (Repository-Methode)

1. **Stacks → Add stack → Repository**
2. **Repository URL:** `https://github.com/basilicom/onyx`
3. **Repository reference:** `refs/heads/basilicom` (Branch — auto-update bei Push)
4. **Compose path:** `deployment/docker_compose/docker-compose.yml`
5. **Additional paths → Add file:** `deployment/docker_compose/docker-compose.basilicom.yml`
6. **Environment variables → Advanced mode** — einfügen (echte Secrets!):

   ```
   IMAGE_TAG=v4.0.7
   TRAEFIK_NETWORK=traefik-proxy
   ONYX_DOMAIN=onyx.example.com
   ONYX_MCP_DOMAIN=onyx-mcp.example.com
   WEB_DOMAIN=https://onyx.example.com
   VPN_SOURCE_RANGE=<VPN-EXIT-IP>/32
   ANTHROPIC_SOURCE_RANGE=160.79.104.0/21
   MCP_BEARER_TOKEN=<secret>
   AUTH_TYPE=disabled
   POSTGRES_PASSWORD=<secret>
   USER_AUTH_SECRET=<secret>
   ENCRYPTION_KEY_SECRET=<secret>
   ```

7. **Deploy**.

### `TRAEFIK_NETWORK`
In dieser Umgebung heißt das Traefik-Netz **`traefik-proxy`**. Das ist der Default
in den Dateien. Falls ihr auf einen anderen Host geht, Wert anpassen.

### Host-Voraussetzung: OpenSearch `vm.max_map_count`
OpenSearch (single-node, 2 GB Heap, `memory_lock`) braucht auf dem Linux-Host:
```bash
sudo sysctl -w vm.max_map_count=262144
# persistent:
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-onyx.conf
```
Fehlt das, crasht der opensearch-Container mit
`max virtual memory areas vm.max_map_count [65530] is too low`. Wenn auf dem
Server schon ein anderer ES/OpenSearch läuft, ist es meist bereits gesetzt.

### RAM
Der Stack braucht **~8–10 GB** (background bis 10 GB Limit, opensearch 2 GB Heap +
2 Model-Server). Für Limits auf einem geteilten Host optional zusätzlich
`-f docker-compose.resources.yml` mergen.

### certresolver
Die Labels nutzen `certresolver=http` — denselben Let's-Encrypt-Resolver-Namen
wie der `bahlsen-demo`-Stack. Falls euer Traefik einen anders benannten Resolver
hat, in `docker-compose.basilicom.yml` anpassen.

## Verifizieren nach Deploy

```bash
# Von außen (sobald Cert ausgestellt):
curl -sI https://onyx.example.com/            # -> 200/3xx
curl -s  https://onyx.example.com/api/health  # -> {"success":true,...}
```

Falls `HTTP 000` / kein Cert: Traefik hat keine Route → `TRAEFIK_NETWORK` falsch,
oder nginx hängt nicht im Traefik-Netz, oder der Cert-Resolver-Name stimmt nicht.

## Nach dem ersten Start (in der UI)

1. Admin-Account anlegen: `https://onyx.example.com/auth/signup`
2. **LLM-Provider** setzen — auf dem Server brauchst du einen erreichbaren:
   OpenRouter / OpenAI (API-Key) **oder** ein Ollama auf dem Server.
   (Hinweis: für ein reines Such-Setup, bei dem Claude antwortet, wird das LLM
   nur für den Setup-Wizard gebraucht.)
3. **Connectors** anlegen: Jira (Projekt `BLSN`), Confluence (Space `BLS`),
   ggf. Google Drive. Atlassian: API-Token + Account-Email.

## Sicherheitsmodell

Die Hosting-Umgebung ist **öffentlich** erreichbar. Onyx hält Jira/Confluence-Daten,
läuft aber mit `AUTH_TYPE=disabled` (kein App-Login) — damit das nicht offen im
Netz liegt, schützen **zwei Traefik-Ebenen**:

1. **IP-Allowlist** (Middleware `ipwhitelist`, Traefik v2):
   - **UI-Router** (`onyx-vpn`): nur `VPN_SOURCE_RANGE` (VPN-Gateway). Mensch-only.
   - **MCP-Router** (`onyx-mcp-allow`): `VPN_SOURCE_RANGE` **+** `ANTHROPIC_SOURCE_RANGE`
     (`160.79.104.0/21`), weil claude.ai sich als Remote-MCP-Client von Anthropics
     Egress aus verbindet.
2. **Bearer-Gate** am MCP-Router: Router matcht nur bei exaktem Header
   `Authorization: Bearer <MCP_BEARER_TOKEN>` — sonst 404. **Das** ist die eigentliche
   Auth für den Anthropic-Pfad (Anthropics Egress teilen sich alle claude.ai-Kunden,
   die IP-Allowlist allein reicht dort nicht). Token stark halten.

> **Folge von `AUTH_TYPE=disabled`:** Jeder im VPN hat Vollzugriff auf die UI
> (inkl. Admin/Connectors). Für ein kleines Team hinter VPN ok. Willst du echtes
> Onyx-Login + per-user-Rechte, brauchst du den Business-Plan (API-Keys) — dann
> `AUTH_TYPE=basic` und der eingebaute MCP-Server mit echten Tokens.

**Verifizieren nach Deploy:**
- im VPN (`curl https://ifconfig.me` == eure VPN-Exit-IP): die Onyx-URL lädt ✓
- ohne VPN: die Onyx-URL → **403** ✓
- MCP ohne/falschem Bearer → **404**, mit korrektem Bearer → MCP-Antwort ✓

## Claude per MCP anbinden

Der eingebaute Onyx-MCP-Server ist in `docker-compose.basilicom.yml` **aktiviert**
(`onyx.mcp_server_main`, Port 8090, Streamable-HTTP, Endpoint `/`) und über
`ONYX_MCP_DOMAIN` (z.B. `onyx-mcp.example.com`) geroutet.

Claude-Client-Konfiguration (HTTP-MCP):
```json
{
  "mcpServers": {
    "onyx": {
      "url": "https://onyx-mcp.example.com/",
      "headers": { "Authorization": "Bearer <MCP_BEARER_TOKEN>" }
    }
  }
}
```
Der Bearer ist derselbe wie `MCP_BEARER_TOKEN` in der Env — Traefik erzwingt ihn.

**Alternative (ohne eingebauten MCP):** stdio-Bridge pro Client gegen
`https://<onyx-domain>/api` mit Session-Login (`/search/send-search-message`).
Sinnvoll, falls ihr `AUTH_TYPE=basic` + echtes Login fahrt.
