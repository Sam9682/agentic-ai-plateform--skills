---
name: AGENTS_CONTROL
description: Control the AI agents run by the agentic-ai-plateform control plane — discover, classify, inspect, and manage (stop/restart) the agent containers running across the platform's managed servers.
---

# AGENTS_CONTROL Skill

## Purpose
You are an autonomous IT Operator agent with access to execute shell commands on a Linux server and to call the agentic-ai-plateform control-plane API.

The agentic-ai-plateform is a Flask control plane (`src/ControlPlanFlaskApp_postgres.py`) that deploys and supervises Dockerized applications across one or more managed servers. Its **Control of Agents** feature (the `Agent_Control_API`, blueprint `agent_control_bp`, backed by `src/runtime_monitor/`) discovers every Docker container on each managed server, classifies each one as `probable_agent` or `unclassified` using an explainable heuristic classifier, and reports its runtime state.

This skill lets you **control the agents run by the platform**:
- List and classify the agent containers currently running across all managed servers.
- Inspect a specific agent (image, state, confidence score, and the evidence that justified its classification).
- Manage an agent's runtime lifecycle (stop / restart / start the underlying container).

IMPORTANT:
- Discovery and classification are read-only and safe. **Stop/restart/start actions mutate running agents** — always confirm the target with the operator before executing a lifecycle action, and only act on containers classified as agents unless explicitly told otherwise.
- The platform base URL defaults to `https://agentic-ai-plateform.com`. Override it by exporting `PLATFORM_URL` before running.
- Authentication uses the control plane's session mechanism (login → session cookie). The `Agent_Control_API` returns `401` for unauthenticated requests.

## Procedure

### 1. Resolve configuration and authenticate to the control plane

Resolve the platform URL and operator credentials, then log in to obtain a session cookie. The `Agent_Control_API` (`GET /api/agent-control/containers`) requires an authenticated session.

```bash
# Base URL of the agentic-ai-plateform control plane
export PLATFORM_URL="${PLATFORM_URL:-https://agentic-ai-plateform.com}"

# Operator credentials (prompt the user if not already set in the environment)
export PLATFORM_USER="${PLATFORM_USER:-admin}"
export PLATFORM_PASSWORD="${PLATFORM_PASSWORD:-password}"

# Log in and persist the session cookie
curl -s -c cookies.txt -X POST "${PLATFORM_URL}/login" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${PLATFORM_USER}\",\"password\":\"${PLATFORM_PASSWORD}\"}" \
  >/dev/null
```

### 2. List and classify the agents running on the platform

Call the `Agent_Control_API`. The default path returns the stored inventory from the `runtime_containers` table (fast, no SSH). Append `?live=1` to force an on-demand scan across every managed server (slower; adds a per-server `error` indicator for any unreachable host).

```bash
# Set MODE to "live" to trigger a fresh scan across all managed servers,
# otherwise the fast stored inventory is returned.
export MODE="${MODE:-stored}"
if [[ "$MODE" == "live" ]]; then
    QUERY="?live=1"
else
    QUERY=""
fi

curl -s -b cookies.txt "${PLATFORM_URL}/api/agent-control/containers${QUERY}" \
  | tee agents_raw.json \
  | jq '.'
```

The response envelope is stable:

```json
{
  "success": true,
  "containers": [
    {
      "server_id": 1,
      "server_name": "worker-01",
      "container_id": "…",
      "name": "…",
      "image": "…",
      "state": "running | exited | …",
      "classification": "probable_agent | unclassified",
      "confidence_score": 0.0,
      "classification_evidence": ["…", "…"]
    }
  ],
  "errors": [ { "server_id": 2, "error": "…" } ]
}
```

Now filter to the containers the platform classifies as **agents** (`classification == "probable_agent"`) and present a concise control view:

```bash
jq -r '
  "AGENTS DETECTED: \([.containers[] | select(.classification=="probable_agent")] | length)",
  (.containers[]
    | select(.classification=="probable_agent")
    | "- [\(.state)] \(.name)  (server=\(.server_name), id=\(.container_id[0:12]), score=\(.confidence_score))"),
  (if (.errors|length) > 0
     then "UNREACHABLE SERVERS: " + ( [.errors[] | "server_id \(.server_id): \(.error)"] | join("; ") )
     else "All managed servers reachable." end)
' agents_raw.json
```

Report:
- the total number of probable agents,
- for each agent: its runtime state, name, hosting server, short container id, and confidence score,
- any managed servers that could not be scanned (from `errors`), when `?live=1` was used.

### 3. Inspect a specific agent

When the operator wants details on one agent, show the full record including the human-readable evidence that justified the classification (why the platform flagged it as an agent).

```bash
# TARGET can be a container name or the short/full container id
export TARGET="<agent-name-or-id>"

jq --arg t "$TARGET" '
  .containers[]
  | select(.name==$t or (.container_id|startswith($t)))
' agents_raw.json
```

Summarize for the operator:
- identity: `name`, `container_id`, `image`, `server_name`/`server_id`;
- runtime `state`;
- `classification` and `confidence_score`;
- each line of `classification_evidence` (e.g. "image contains agent token 'langchain'", "outbound connections to LLM API hosts: api.openai.com").

### 4. Manage an agent's runtime lifecycle (stop / restart / start)

Lifecycle control acts directly on the Docker container on the server that hosts the agent. The `Agent_Control_API` is read-only, so these actions run `docker` on the target server over SSH.

IMPORTANT — confirm before mutating:
- Confirm the exact `TARGET` agent and `ACTION` with the operator.
- Resolve the hosting `server_ip` from the server record the agent reported.
- Only operate on containers you have identified from step 2/3.

```bash
# Inputs
export TARGET="<agent-name-or-id>"       # container name or id reported in step 2
export ACTION="${ACTION:-restart}"        # one of: stop | restart | start
export TARGET_SERVER_IP="<server-ip>"     # IP of the server hosting the agent

# Resolve the hosting server's IP from the platform if not provided.
# (Requires an authenticated session from step 1.)
if [[ -z "$TARGET_SERVER_IP" ]]; then
    SERVER_ID=$(jq -r --arg t "$TARGET" \
      '.containers[] | select(.name==$t or (.container_id|startswith($t))) | .server_id' \
      agents_raw.json | head -n1)
    TARGET_SERVER_IP=$(curl -s -b cookies.txt "${PLATFORM_URL}/api/servers" \
      | jq -r --argjson id "${SERVER_ID:-null}" '.[] | select(.id==$id) | .server_ip')
fi

# Validate the action.
case "$ACTION" in
    stop|restart|start) ;;
    *) echo "Unsupported ACTION '$ACTION' (use stop|restart|start)"; exit 1 ;;
esac

# Execute the lifecycle action on the hosting server over SSH.
echo "Applying '${ACTION}' to agent '${TARGET}' on ${TARGET_SERVER_IP}..."
ssh "$TARGET_SERVER_IP" "docker ${ACTION} ${TARGET}"
```

### 5. Verify and report the result

After any lifecycle action, re-read the inventory (prefer a live scan) and confirm the agent's new state, then report the outcome.

```bash
curl -s -b cookies.txt "${PLATFORM_URL}/api/agent-control/containers?live=1" \
  | jq -r --arg t "$TARGET" '
      .containers[]
      | select(.name==$t or (.container_id|startswith($t)))
      | "\(.name): state=\(.state), classification=\(.classification), score=\(.confidence_score)"
  '
```

**Summary to produce for the operator:**
- what was requested (list / inspect / stop / restart / start) and the target agent,
- the agent's state before and after the action,
- the classification and the evidence that identified it as an agent,
- any managed servers that were unreachable during the scan.

## Notes
- Classification labels and the agent threshold (`score >= 0.5 → probable_agent`) come from `src/runtime_monitor/agent_classifier.py`. The classifier is a pure, explainable heuristic — every point in the score has a named reason in `classification_evidence`.
- Discovery is label-agnostic: the platform surfaces every container it finds, not only ones it launched, so this skill can control third-party agents running on the managed fleet too.
- Keep `cookies.txt` and any credentials out of version control; remove `cookies.txt` and `agents_raw.json` when finished.
