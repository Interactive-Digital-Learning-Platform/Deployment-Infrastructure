# Deployment infrastructure

This directory contains infrastructure containers and the Nginx API gateway.
The backend services are intentionally not containerized. Run the infrastructure
from this directory:

```bash
docker compose up -d
```

Nginx reaches backend processes running on the Docker host through
`host.docker.internal`. Start each backend on its assigned port and bind it to an
address reachable from Docker (for example, Uvicorn's `--host 0.0.0.0`).

## API gateway routes

The gateway is available at `http://localhost:8080` and removes the service
prefix before forwarding each request.

| Gateway prefix | Host-run backend service | Temporary port |
| --- | --- | --- |
| `/api/pdf/` | PDF ingestion | `8001` |
| `/api/assistant/` | AI learning assistant | `8002` |
| `/api/quiz/` | Personalized quiz | `8003` |
| `/api/notes/` | Handwritten notes | `8004` |
| `/api/battle/` | Quiz-Battle-Service (1v1 battle) | `8005` |
| `/api/mcp/` | Learning Assistant MCP server | `8006` |

Examples:

- `/api/pdf/ingest/...` forwards to the PDF service's `/ingest/...` route.
- `/api/assistant/conversations/...` forwards to `/conversations/...`.
- `/api/quiz/api/v1/quiz/...` forwards to `/api/v1/quiz/...`.
- `/api/notes/api/health` forwards to the notes service's `/api/health`.
- `/api/battle/api/v1/battle/...` forwards to Quiz-Battle-Service's `/api/v1/battle/...`
  (REST and the `WS /api/v1/battle/match/{id}/ws` realtime endpoint alike).
- `/api/mcp/mcp` forwards to the MCP server's Streamable HTTP endpoint `/mcp`
  (a trailing slash, `/api/mcp/mcp/`, is 308-redirected to the no-slash form);
  `/api/mcp/health` forwards to `/health`. Callers send
  `Authorization: Bearer <MCP_AUTH_TOKEN>`. The `/api/mcp/` location keeps
  responses unbuffered and allows 1h idle for long-lived MCP sessions.
- `/learning-documents/...` proxies straight to MinIO (`9000`) with the path
  preserved, so the MCP server can hand the mobile app presigned PDF download
  URLs on this origin in local dev. The bucket is private; only a valid SigV4
  signature works. In production the MCP server signs against Cloudflare R2's
  public endpoint and the gateway is not involved.
- `/health` is the API gateway's own health endpoint.

Infrastructure ports copied from the PDF ingestion Compose setup remain
available: Redis `6379`, Qdrant `6333`/`6334`, and MinIO `9000`/`9001`.

The backend `.env` files must use host-accessible infrastructure endpoints when
the services run outside Docker, such as `localhost:6379`, `localhost:6333`, and
`localhost:9000`.
