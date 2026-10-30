---
name: KILL
description: Forcefully kill all running Docker containers of a deployed application instance for a specific user, immediately terminating the processes (SIGKILL) instead of a graceful shutdown, then remove the stopped containers while preserving volumes.
---

# KILL Skill

## Purpose
You are an autonomous Operations agent running on a Linux server with access to
the local filesystem and shell commands.

This skill **forcefully kills** all running Docker containers that belong to a
deployed application instance for a given user. Unlike the `STOP_APP` skill — which
performs a graceful `docker stop` (SIGTERM then timeout) — this skill issues a
`docker kill` (SIGKILL) to terminate the containers **immediately**, then removes
the dead containers. Use it when the application is unresponsive, hung, or must be
torn down right away.

The application source code and its `docker-compose` setup are located in the
git repository:

```
REPO_DIR = "{{APPLICATION_FOLDER}}"
```

IMPORTANT:
- All commands have to be executed in the application located `{{APPLICATION_FOLDER}}`.
- Execute all steps with the environment variable `USER_ID={{USER_ID}}`.
- `docker kill` is **not graceful**: in-flight requests are dropped and the
  application has no chance to flush state. **Named volumes and data are
  preserved** (we never run `docker volume rm`), but any unsaved in-memory state
  is lost. Only use KILL when an immediate termination is required.

## Procedure

### 1. Calculate HTTP Ports

Calculate the HTTP/HTTPS ports used by the Docker containers of the application.
Use the following command:

```bash
source {{APPLICATION_FOLDER}}/conf/deploy.ini
RANGE_START=${RANGE_START:-6000}
RANGE_RESERVED=${RANGE_RESERVED:-100}
RANGE_PORTS_PER_APPLICATION=${RANGE_PORTS_PER_APPLICATION:-12}
if ! [[ "$USER_ID" =~ ^[0-9]+$ ]]; then
    USER_ID=0
fi
PORT_NAMES=(
    HTTPS_PORT HTTP_PORT
    HTTPS_PORT1 HTTP_PORT1
    HTTPS_PORT2 HTTP_PORT2
    HTTPS_PORT3 HTTP_PORT3
    HTTPS_PORT4 HTTP_PORT4
    HTTPS_PORT5 HTTP_PORT5
)
PORT_RANGE_BEGIN=$((RANGE_START + USER_ID * RANGE_RESERVED))
base=$((PORT_RANGE_BEGIN + APPLICATION_IDENTITY_NUMBER * RANGE_PORTS_PER_APPLICATION))
offset=0
for name in "${PORT_NAMES[@]}"; do
    printf -v "$name" '%s' "$((base + offset))"
    export "$name"
    offset=$((offset + 1))
done
```

### 2. Identify the target containers

Find the running containers that belong to this application instance, based on
the naming convention used at deploy time (name pattern
`${NAME_OF_APPLICATION}-.*-${USER_ID}-.*`):

```bash
echo "Identifying {{APPLICATION_NAME}} containers for USER_ID=$USER_ID..."
containers=$(docker ps -q --filter "name=${NAME_OF_APPLICATION}-.*-${USER_ID}-.*")

if [[ -z "$containers" ]]; then
    echo "No running containers found for USER_ID: $USER_ID — nothing to kill."
    exit 0
fi

echo "The following containers will be KILLED (SIGKILL):"
docker ps --filter "name=${NAME_OF_APPLICATION}-.*-${USER_ID}-.*" \
    --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}'
```

### 3. Forcefully kill the containers

Send SIGKILL to every matching container to terminate it immediately, then
remove the dead containers. **Do not** run a graceful `docker stop` first — the
point of this skill is the forced kill.

```bash
echo "Killing {{APPLICATION_NAME}} services (SIGKILL)..."

# Force immediate termination of the processes inside the containers.
docker kill $containers

# Remove the now-dead containers (force in case any lingers).
docker rm -f $containers

echo "Services killed successfully ✅"
```

### 4. Verify and report

After completion, verify the result and report:

```bash
remaining=$(docker ps -q --filter "name=${NAME_OF_APPLICATION}-.*-${USER_ID}-.*")
if [[ -z "$remaining" ]]; then
    echo "✅ All {{APPLICATION_NAME}} containers for USER_ID=$USER_ID are gone."
else
    echo "⚠️  Some containers are still running:"
    docker ps --filter "name=${NAME_OF_APPLICATION}-.*-${USER_ID}-.*" \
        --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}'
fi
```

**After completion, verify:**
- All targeted containers have been killed and removed.
- Named volumes are **preserved** (data is not deleted).
- Any dangling networks created only for this instance may remain and are safe to ignore.

**Summary:** print a short summary including:
- The `USER_ID` and application name targeted.
- The list of containers that were killed (IDs / names).
- The final status (all killed, or any containers still running with the reason).
- Any warnings (e.g. no containers were found, a container could not be removed).

If ANY step fails, explain clearly which step failed and why.
