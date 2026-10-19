# KurtAPI developer documentation

KurtAPI is a hosted gateway for the ER:LC API. Your application calls `api.thekarti.com` with a KurtAPI key. KurtAPI checks the key's permissions, selects a fixed ER:LC operation, and performs the upstream request. Your application does not need an ER:LC master key, Global API key, or direct ER:LC API access.

## Get started

1. Obtain a KurtAPI key from the KurtAPI dashboard. The key is displayed once when created; store it as a server-side secret.
2. Install a client library:

   ```sh
   pip install kurtapi-client
   npm install @thekarticompany/kurtapi
   ```

3. Use the key from an environment variable:

   ```sh
   KURTAPI_API_KEY=your-key
   ```

Python:

```python
import os
from kurtapi import KurtAPI

with KurtAPI(os.environ["KURTAPI_API_KEY"]) as api:
    status = api.status()
    print(status)
```

Node.js:

```js
const { KurtAPI } = require("@thekarticompany/kurtapi");

const api = new KurtAPI({ apiKey: process.env.KURTAPI_API_KEY });
const status = await api.status();
console.log(status);
```

The package names are the intended public names and must be registered under the KurtAPI publisher accounts before release. Do not install from these names until their first official release is announced.

## Authentication

Every SDK request sends `Authorization: Bearer <KurtAPI key>` over HTTPS. Keys are bound to one ER:LC server and have a fixed permission set. Do not expose a key in browser code, mobile app bundles, source control, or logs. Create separate keys for separate applications and grant only the permissions each application needs.

The SDKs target `https://api.thekarti.com` by default. A custom `baseUrl` is supported for development or an explicitly configured compatible deployment.

## Permissions

The exact permission catalog and command names are maintained in [`../../perm_keys.md`](../../perm_keys.md). Current operation-to-permission mapping:

| Operation | Required permission | Input | Result |
| --- | --- | --- | --- |
| `status.read` | `status.read` | none | ER:LC server information |
| `players.read` | `players.read` | none | `Players` data |
| `staff.read` | `staff.read` | none | `Staff` data |
| `join_logs.read` | `join_logs.read` | none | `JoinLogs` data |
| `queue.read` | `queue.read` | none | `Queue` data |
| `kill_logs.read` | `kill_logs.read` | none | `KillLogs` data |
| `command_logs.read` | `command_logs.read` | none | `CommandLogs` data |
| `moderator_calls.read` | `moderator_calls.read` | none | `ModCalls` data |
| `bans.read` | `bans.read` | none | Ban list |
| `vehicles.read` | `vehicles.read` | none | `Vehicles` data |
| `emergency_calls.read` | `emergency_calls.read` | none | `EmergencyCalls` data |
| `command.execute` | One of the command execution permissions | `{ "command": ":..." }` | ER:LC command response |

Command permissions are hierarchical: administrator includes moderator commands; owner includes administrator and moderator commands. The server validates command names and access before forwarding them. Unknown commands are not forwarded.

Only operations registered by the server are available. Do not assume that a route or operation exists because the upstream ER:LC API supports it.

## ER:LC upstream API version

KurtAPI forwards server-information reads and command execution through ER:LC API v2. The `status.read` operation uses `GET /v2/server`; the other information operations request their optional fields through v2 query parameters. `bans.read` currently uses ER:LC's v1 bans endpoint because ER:LC's v2 API reference does not document a bans equivalent. KurtAPI's own public endpoint remains `POST /v1/operations/:operation`; that version number is independent of the ER:LC upstream API version. ER:LC's [large application requirements](https://apidocs.erlc.gg/large-applications#requirements) call for full v2 use while allowing v1 APIs for functionality not available in v2.

## Raw HTTP

The public route is `POST /v1/operations/:operation` in [`src/server.js`](../../src/server.js). The `:operation` path segment must exactly match a key registered in [`src/operations.js`](../../src/operations.js), such as `players.read` or `command.execute`. Requests use a KurtAPI Bearer key and a JSON body; read operations use `{}`, while command execution uses `{"command":"..."}`.

The examples use the public base URL configured as `publicBaseUrl` in the deployment's `config.json` (`https://api.thekarti.com` for the hosted service). For another deployment, replace that host with its public base URL; keep the `/v1/operations/...` path shown here.

```sh
curl -sS https://api.thekarti.com/v1/operations/players.read \
  -H "Authorization: Bearer $KURTAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{}'
```

```sh
curl -sS https://api.thekarti.com/v1/operations/command.execute \
  -H "Authorization: Bearer $KURTAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command":":h Hello from KurtAPI"}'
```

## Errors and status codes

For errors the API returns a JSON object with an `error` field. The SDKs raise a `KurtAPIError` (Python) or `KurtAPIError` (Node.js) for non-2xx responses. The exception includes the HTTP status and parsed response body when available.

| HTTP | Typical error | Meaning |
| --- | --- | --- |
| `400` | `invalid_request_body` | Invalid operation input |
| `401` | `missing_api_key`, `invalid_or_revoked_api_key`, `api_key_expired` | Key is missing, invalid, deleted, or expired |
| `403` | `insufficient_permissions` | The key lacks permission for this operation or command |
| `404` | `unknown_operation`, `command_not_found` | Operation or command is not registered; command response may include permitted suggestions |
| `429` | rate limit response | Too many requests; respect any `Retry-After` header |
| `502` | `invalid_upstream_response` | ER:LC returned a response KurtAPI could not use |
| `503` | `upstream_application_key_not_configured` | KurtAPI upstream configuration problem |
| `504` | `upstream_timeout` | ER:LC did not respond before the gateway timeout |
| `5xx` | varies | KurtAPI or ER:LC service error |

ER:LC response status codes and bodies are relayed by the gateway where possible. Check the status and body even when the HTTP exchange itself succeeded. SDKs do not automatically retry commands, because repeating a command can cause duplicate in-game actions. Retry read operations only when appropriate for your application.

## Current limitations

- The server's registered operation list is authoritative and can differ from the full ER:LC API.
- Operations currently return JSON; SDK response payloads are not yet modeled as stable typed objects.
- KurtAPI is not affiliated with or endorsed by ER:LC or PRC.

## References

- [Python SDK](../../packages/python/README.md)
- [Node.js SDK](../../packages/node/README.md)
- [Permission catalog](../../perm_keys.md)
- [ER:LC API docs](https://apidocs.erlc.gg/)
