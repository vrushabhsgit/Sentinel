# Sentinel

**Permission-enforced support desk over MCP.**

One chat interface. Multiple roles. Every action is decided by the scopes carried in the caller’s JWT and enforced by the MCP server. The model never sees a tool it is not allowed to call. The database never returns a row the caller is not allowed to see.

---

## What it does

A course platform needs support. Learners ask about access, payments, certificates and technical issues. Support agents work the queue, look up accounts and extend access. Admins manage certificates, FAQs, statistics and users.

Sentinel collapses the usual set of admin screens into a single chat endpoint. The same endpoint is used by every role. Authorization is not implemented in the UI, the system prompt, or the model. It is implemented once, in the MCP server, using scopes carried inside a JWT.

| Role    | Capabilities |
|---------|--------------|
| learner | Own enrollments, payments, certificates and tickets. Purchase courses. Raise and reply on own tickets. |
| support | Full ticket queue (including internal notes), learner lookup, access extension (capped), enrollment removal. |
| admin   | Everything support can do, plus certificate issuance, FAQ management, statistics and user administration. |

---

## Quick start

```bash
# Create and seed the database
uv run python -m database.db

# Start the MCP server
uv run python -m mcp_server.server

# Start the API + UI
uv run python -m backend.app
```

Open `http://127.0.0.1:8000`.

Generate a JWT secret and put it in `.env`:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

All demo accounts use the password `password123`.

---

## Demo accounts

| Email                 | Role    | Starting state |
|-----------------------|---------|----------------|
| rahul@learner.com     | learner | Java completed (no certificate issued), failed Agentic AI payment |
| priya@learner.com     | learner | Spring Boot access expired, Java at 25% |
| aman@learner.com      | learner | DevOps purchased four days ago |
| neha@telusko.com      | support | Ticket queue |
| vikram@telusko.com    | support | Ticket queue |
| akshay@telusko.com    | admin   | Full access |

---

## Architecture

Three processes, two ports.

```
browser  →  backend :8000  →  mcp server :8001
         login / chat          tools/list + tools/call
         (holds JWT)           (same JWT forwarded)
```

1. Login returns a short-lived JWT containing the user’s identity and effective scopes.
2. The chat endpoint opens a fresh MCP client per request and forwards the caller’s JWT as the Bearer token.
3. MCP middleware inspects every `tools/list` and `tools/call`. Unauthorized tools are filtered from the list; unauthorized calls are rejected before the tool body runs.
4. The model runs a bounded tool-calling loop (maximum 8 rounds). Results are returned to the client together with a trace.

The browser never talks to the MCP server. There is no service account and no elevated fallback.

---

## Core properties

- **Scopes travel in the token.** Permissions are plain `resource:action` strings. Custom scopes can override role defaults. An empty scopes field means the user has no permissions.
- **Middleware is the authorization boundary.** Filtering `tools/list` improves the model’s context. The actual security check is on `tools/call`. Knowing a tool name is not sufficient to invoke it.
- **Row-level rules live in the tools.** A learner can call `get_ticket` but only for their own tickets. Internal notes are excluded unless the token carries `tickets:read_all`.
- **Limits inside a shared scope.** Support and admin both hold `access:extend`. Only admin holds `access:extend_unlimited`. The 30-day cap is enforced in code, not by prompt.
- **Destructive tools require confirmation of identity.** `purchase_course` and `remove_learner_from_course` demand both the course id and a matching title. The model cannot invent an id from earlier context.
- **Identity comes from the verified token.** “My” tools read the user id from `current_user()`. The model never receives or supplies a user id.
- **One MCP client per request.** The session is authenticated as the current user and is closed when the request ends.
- **Trace is first-class.** Every tool call (name, arguments, result, error flag) is returned so failures can be diagnosed without guessing.
- **SQLite by design for this deployment.** No external database server is required. The schema, models and tools are written so that moving to Postgres is a change to a single connection module.

---

## Tech stack

| Component     | Implementation |
|---------------|----------------|
| Language      | Python |
| API           | FastAPI |
| MCP server    | MCP SDK (`mcp>=2.3`), streamable HTTP, custom `JWTTokenVerifier`, middleware |
| Authentication| JWT (HS256), issuer + audience validation, required claims |
| Authorization | Scope checks in middleware + row-level checks inside tools |
| Database      | SQLite (single file) |
| Data access   | Dataclass models; all SQL lives in the model layer |
| LLM           | OpenAI (default `gpt-4o-mini`) |
| Frontend      | Single static HTML file |
| Tooling       | uv |

---

## Layout

```
common/
  config.py          # settings, JWT parameters, ports
  permissions.py     # role → scope mapping, effective_scopes()
  auth.py            # create_token, decode_token
  prompts.py         # system prompt, server instructions
database/
  schema.sql
  seed.sql
  db.py              # connection, foreign keys, reset
  models.py          # one class per table
mcp_server/
  server.py          # TOOL_SCOPES, middleware, tools, token verifier
backend/
  app.py             # login, chat loop, admin endpoints
frontend/
  index.html
```

---

## Scopes

**Learner**  
`faq:read` `courses:read` `courses:purchase` `me:read` `tickets:read` `tickets:create` `tickets:reply`

**Support**  
Learner scopes that are not personal, plus:  
`tickets:read_all` `tickets:manage` `learners:read` `access:extend` `enrollments:remove`

**Admin**  
All support scopes, plus:  
`access:extend_unlimited` `certificates:issue` `faq:write` `stats:read` `users:manage`

Admin is defined as support plus a delta, so any new support capability is automatically available to admin.

---

## Design notes

- Authentication (is this a valid token issued by us for this audience?) and authorization (does this token contain the required scope?) are separate gates.
- The model is instructed to attempt the requested action. It does not pre-refuse. The server is the source of truth.
- Tool descriptions are written for the model. When a tool is chosen incorrectly, the docstring is the first place to revise.
- Role, category and priority values currently live in SQL `CHECK` constraints. They belong in configuration; the current placement is intentional technical debt.
- Token scopes are snapshotted at login. Role changes take effect on the next login. Existing tokens remain valid until expiry.

---

## License

MIT
