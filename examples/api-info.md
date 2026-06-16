# DAP Orchestrator API (curl) how-to

A practical, copy-paste guide to driving the DAP Orchestrator with **`curl`**. 
It covers: getting and using an auth token; creating/updating/deleting secrets;
uploading a blueprint; deploying it; checking status; gracefully uninstalling and deleting a deployment; 
and reading an endpoint's available network segments and datastores.

All examples use shell variables defined once below — **fill in your own values**
(nothing here is a real credential or URL).

---

## 0. Set up your variables

```bash
# --- Orchestrator + portal endpoints (no trailing slash) ---
export ORCH="https://orchestrator.<env>.edge.tme"   # orchestrator API host
export PORTAL="https://portal.<env>.edge.tme"       # portal host (issues tokens)

# --- OAuth2 client credentials (from the orchestrator's API client) ---
export ORG_ID="<org-id-uuid>"
export CLIENT_ID="<orchestrator-client-id>"
export CLIENT_SECRET="<orchestrator-client-secret>"
```

> **Self-signed certificates:** lab orchestrators usually present a self-signed
> TLS cert, so every `curl` below uses `-k` (a.k.a. `--insecure`). Drop it if
> your orchestrator has a trusted cert.

> **JSON parsing:** the examples pipe responses through `python3` (always
> available) to pull out a field. `jq` works too if you have it — e.g.
> `... | jq -r '.access_token'`.

---

## 1. Get and use a token

Tokens come from the **portal** OIDC endpoint; you then send them to the
**orchestrator**.

```bash
export TOKEN=$(curl -sk -X POST "$PORTAL/rest/v1/oidc/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "org_id=$ORG_ID" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "token length: ${#TOKEN}"   # sanity check — should be a few hundred chars
```

Best practices / gotchas:

- **No `Bearer` prefix.** The orchestrator expects the raw token:
  `-H "Authorization: $TOKEN"` (NOT `Authorization: Bearer ...`).
- **Token lifetime is ~6 hours.** Re-run the request above when calls start
  returning `401`. A 401 with `failed to validate client credentials` means the
  token is missing/expired (or the client secret was rotated).
- **Get the token from `$PORTAL`, send it to `$ORCH`.** Requesting a token
  straight from the orchestrator host typically fails.

Quick check that the token works:

```bash
curl -sk "$ORCH/rest/v1/blueprints?_size=1" -H "Authorization: $TOKEN" \
  -o /dev/null -w "HTTP %{http_code}\n"        # expect 200
```

---

## 2. Secrets

Secrets live at `"$ORCH/rest/v1/secrets/<name>"`. A blueprint input only ever
references a secret **by name**; the value is resolved at deploy time. Sensitive
fields are **write-only** — a `GET` returns them masked as `***`.

See `secrets-info.md` for which schema each blueprint input expects.

### 2a. Create a `password` secret (simple string value)

```bash
curl -sk -X POST "$ORCH/rest/v1/secrets/cma-vm-pw" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"password","value":"<the-password>"}' \
  -w "\nHTTP %{http_code}\n"                    # expect 201
```

### 2b. Create a `binary_configuration` secret (image/binary download)

```bash
curl -sk -X POST "$ORCH/rest/v1/secrets/cma-win-image" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "type": "binary_configuration",
        "value": {
          "binary_image_url": "https://<artifact-host>/win-server-2022.qcow2",
          "binary_image_version": "1.0.3",
          "binary_image_access_user": "<user>",
          "binary_image_access_token": "<token>"
        }
      }' \
  -w "\nHTTP %{http_code}\n"                    # expect 201
```

> `binary_image_url` and `binary_image_version` are required;
> `binary_image_access_user` / `binary_image_access_token` are only needed when
> the artifact host requires authentication.

### 2c. Create a `basic_auth_credentials` secret (username + password)

```bash
curl -sk -X POST "$ORCH/rest/v1/secrets/cma-smtp-cred" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"basic_auth_credentials","value":{"username":"<user>","password":"<password>"}}' \
  -w "\nHTTP %{http_code}\n"                    # expect 201
```

### 2d. Read a secret (value masked)

```bash
curl -sk "$ORCH/rest/v1/secrets/cma-win-image" -H "Authorization: $TOKEN" \
  | python3 -m json.tool
# -> "binary_image_access_token": "***"   (sensitive fields are never returned)
```

### 2e. Update a secret (`PATCH`, deep-merge on `value`)

`PATCH` merges into the existing `value`, so you can change one key without
re-sending the rest. Handy for bumping `binary_image_version` to force the
endpoint to re-download an image whose contents changed:

```bash
curl -sk -X PATCH "$ORCH/rest/v1/secrets/cma-win-image" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value":{"binary_image_version":"1.0.4"}}' \
  -w "\nHTTP %{http_code}\n"                    # expect 200
# binary_image_url / access_* are preserved; only the version changes.
```

### 2f. Delete a secret

```bash
curl -sk -X DELETE "$ORCH/rest/v1/secrets/cma-win-image" \
  -H "Authorization: $TOKEN" -w "HTTP %{http_code}\n"   # expect 204
```

---

## 3. Upload a blueprint

Blueprint archive upload uses the `"$ORCH/api/v3.1/blueprints/<id>"` endpoint
(multipart). The `<id>` is the name the blueprint will have on the orchestrator.

**Package first.** The archive (`.zip` or `.tar.gz`) must contain a **single
top-level directory** with the blueprint's main YAML at its root:

```
MY_WindowsServer.tar.gz
└── MY_WindowsServer/
    ├── MY_WindowsServer.yaml        <- main file (the entrypoint)
    ├── infrastructure/ ...
    └── ...
```

```bash
# from the directory that CONTAINS the MY_WindowsServer/ folder:
tar -czf /tmp/MY_WindowsServer.tar.gz MY_WindowsServer
#   or:  zip -r /tmp/MY_WindowsServer.zip MY_WindowsServer
```

**Upload.** `application_file_name` tells the orchestrator which file inside the
archive is the entrypoint (it does not have to match the `<id>`):

```bash
curl -sk -X PUT "$ORCH/api/v3.1/blueprints/MY_WindowsServer" \
  -H "Authorization: $TOKEN" \
  -F 'params={"application_file_name":"MY_WindowsServer.yaml","visibility":"tenant"}' \
  -F "blueprint_archive=@/tmp/MY_WindowsServer.tar.gz" \
  -w "\nHTTP %{http_code}\n"                    # expect 201
```

Parsing/validation is asynchronous. Confirm it finished cleanly:

```bash
curl -sk "$ORCH/rest/v1/blueprints/MY_WindowsServer" -H "Authorization: $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('state:', d.get('state'))"
# -> state: uploaded   (anything else -> inspect the response for an error)
```

> **Re-uploading:** you cannot overwrite an existing revision (you'll get
> `409`). Either upload a new revision, or delete the blueprint first (see 3a).
> You also cannot change `application_file_name` on an existing blueprint
> without deleting it first.

### 3a. Delete a blueprint

A blueprint with deployments cannot be deleted (`409`) — delete those
deployments first (section 5).

```bash
curl -sk -X DELETE "$ORCH/api/v3.1/blueprints/MY_WindowsServer" \
  -H "Authorization: $TOKEN" -w "HTTP %{http_code}\n"   # expect 204
```

---

## 4. Deploy a blueprint

Creating a deployment both creates **and installs** it. Use
`"$ORCH/rest/v1/deployments/<deployment-id>"` (POST). The body has three parts:

- `blueprint_id` — which blueprint to deploy.
- `inputs` — the deployment inputs (see each `*-input-example.json`).
- `tags` — must include **`csys-obj-parent`**, which binds the deployment to a
  parent environment:
  - **standalone VM blueprints** bind to the **endpoint**:
    `ece-<endpoint_service_tag>`
  - **layered app blueprints** bind to the **base deployment id**:
    e.g. `cma-base-01`

### 4a. Deploy with an inline payload

```bash
curl -sk -X POST "$ORCH/rest/v1/deployments/cma-base-01" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "blueprint_id": "MY_WindowsServer",
        "inputs": {
          "vm_user_name": "Administrator",
          "vm_password_secret_name": "cma-vm-pw",
          "artifact_configuration_secret_name": "cma-win-image",
          "hostname": "cma-base-01",
          "disk": "/DataStore0",
          "segment_name": "bridge0",
          "dhcp": true,
          "enable_winrm": true
        },
        "tags": [ { "key": "csys-obj-parent", "value": "ece-<endpoint_service_tag>" } ]
      }' \
  -w "\nHTTP %{http_code}\n"
```

### 4b. Deploy from an input JSON file (cleaner for big input sets)

Put the whole body in a file and reference it with `-d @file`:

```bash
cat > /tmp/deploy-fileserver.json <<'JSON'
{
  "blueprint_id": "MY_File_Server",
  "inputs": {
    "imo_number": "9876543",
    "default_password_secret_name": "cma-vm-pw",
    "quota_multiplier": 1.0
  },
  "tags": [ { "key": "csys-obj-parent", "value": "cma-base-01" } ]
}
JSON

curl -sk -X POST "$ORCH/rest/v1/deployments/cma-fileserver-01" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/deploy-fileserver.json \
  -w "\nHTTP %{http_code}\n"
```

> Tip: the `inputs` object is exactly the JSON form of a
> `*-input-example.json` (strip the `//` comments first). Only set the inputs
> you care about; the rest take blueprint defaults.

---

## 5. Track, uninstall, and delete a deployment

### 5a. Check status

```bash
curl -sk "$ORCH/rest/v1/deployments/cma-base-01" -H "Authorization: $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
print('state:', d.get('state'), '| status:', d.get('deployment_status'), '| error:', d.get('error'))"
```

Useful values:

| Field               | Meaning                                                              |
| ------------------- | ------------------------------------------------------------------- |
| `state`             | Workflow state: `pending` / `started` / `completed` / `failed`      |
| `deployment_status` | Lifecycle: `installing` / `deployed` / `uninstalling` / `uninstalled` |
| `error`             | Failure detail (empty when healthy)                                 |

A finished, healthy deploy is `state=completed`, `deployment_status=deployed`.

Poll until it settles:

```bash
while true; do
  s=$(curl -sk "$ORCH/rest/v1/deployments/cma-base-01" -H "Authorization: $TOKEN" \
      | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('state'), d.get('deployment_status'))")
  echo "$s"; echo "$s" | grep -qE 'completed|failed' && break; sleep 30
done
```

### 5b. Read the deployment outputs (capabilities)

```bash
curl -sk "$ORCH/rest/v1/deployments/cma-base-01/capabilities" \
  -H "Authorization: $TOKEN" | python3 -m json.tool
# e.g. MY_Base_Host_IP, Virtual_Machine_Hostname, ...
```

### 5c. Gracefully uninstall, then delete

Always **uninstall before deleting** — uninstall is what actually removes the
VM(s). Deleting a deployment that still has live nodes is rejected.

```bash
# 1) start the uninstall workflow
curl -sk -X POST "$ORCH/rest/v1/executions" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"deployment_id":"cma-base-01","workflow_id":"uninstall"}' \
  -w "\nHTTP %{http_code}\n"

# 2) wait for it to finish (state=completed, deployment_status=uninstalled)
while true; do
  s=$(curl -sk "$ORCH/rest/v1/deployments/cma-base-01" -H "Authorization: $TOKEN" \
      | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('state'), d.get('deployment_status'))")
  echo "$s"; echo "$s" | grep -qiE 'completed uninstalled|failed' && break; sleep 20
done

# 3) delete the (now-uninstalled) deployment record
curl -sk -X DELETE "$ORCH/rest/v1/deployments/cma-base-01" \
  -H "Authorization: $TOKEN" -w "HTTP %{http_code}\n"     # expect 200/202/204
```

> **Order matters for layered apps:** uninstall+delete the layered children
> (e.g. `cma-fileserver-01`, `cma-v2ps-01`) **before** their base
> (`cma-base-01`). Don't force-delete — that skips uninstall and can orphan VMs
> that have no API to remove.

---

## 6. Inspect an endpoint (network segments & datastores)

When filling in `segment_name` and `disk`, you need the names the endpoint
actually offers. The full hardware inventory is on the endpoint object — but you
must fetch it **by UUID** (the by-name lookup returns a summary only).

**Step 1 — resolve the endpoint UUID from its service tag:**

```bash
EP_ID=$(curl -sk "$ORCH/rest/v1/endpoints?filter=service_identifier%20eq%20<service-tag>" \
  -H "Authorization: $TOKEN" \
  | python3 -c "import sys,json; r=json.load(sys.stdin).get('results',[]); print(r[0]['id'] if r else '')")
echo "$EP_ID"
```

**Step 2 — list the bridge/NAT network segments** (these are valid
`segment_name` values):

```bash
curl -sk "$ORCH/rest/v1/endpoints/$EP_ID" -H "Authorization: $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
[print(s['name'],'->',s['type']) for s in d.get('network_virtual_segments',[]) if s.get('type') in ('BRIDGE','NAT')]"
# e.g.  bridge0 -> BRIDGE
```

**Step 3 — list the online datastores** (these are valid `disk` values):

```bash
curl -sk "$ORCH/rest/v1/endpoints/$EP_ID" -H "Authorization: $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
[print(v['name'],'->',v['status'],v.get('free','')+' free') for v in d.get('hardware_storage_volumes_data_stores',[])]"
# e.g.  /DataStore0 -> ONLINE 3518.12 GB free
```

The same endpoint object also carries CPU/memory, NICs, GPUs, USB/serial/PCI
peripherals, and storage controllers/disks if you need them — inspect the full
JSON with `curl -sk "$ORCH/rest/v1/endpoints/$EP_ID" -H "Authorization: $TOKEN" | python3 -m json.tool`.

---

## Endpoint / path quick reference

| Action                         | Method & path                                                  |
| ------------------------------ | -------------------------------------------------------------- |
| Get token                      | `POST $PORTAL/rest/v1/oidc/token`                              |
| Create / read / delete secret  | `POST` / `GET` / `DELETE $ORCH/rest/v1/secrets/<name>`         |
| Update secret (merge)          | `PATCH $ORCH/rest/v1/secrets/<name>`                           |
| Upload / delete blueprint      | `PUT` / `DELETE $ORCH/api/v3.1/blueprints/<id>`                |
| Get blueprint status           | `GET $ORCH/rest/v1/blueprints/<id>`                            |
| Create (install) deployment    | `POST $ORCH/rest/v1/deployments/<id>`                          |
| Get deployment status          | `GET $ORCH/rest/v1/deployments/<id>`                           |
| Get deployment outputs         | `GET $ORCH/rest/v1/deployments/<id>/capabilities`              |
| Uninstall (run workflow)       | `POST $ORCH/rest/v1/executions` (`workflow_id: uninstall`)     |
| Delete deployment              | `DELETE $ORCH/rest/v1/deployments/<id>`                        |
| Find endpoint UUID by tag      | `GET $ORCH/rest/v1/endpoints?filter=service_identifier eq <tag>` |
| Endpoint hardware inventory    | `GET $ORCH/rest/v1/endpoints/<uuid>`                           |

All orchestrator calls send `-H "Authorization: $TOKEN"` (no `Bearer` prefix)
and, against a self-signed cert, `-k`.
