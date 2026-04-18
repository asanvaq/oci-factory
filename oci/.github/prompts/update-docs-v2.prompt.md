---
description: "Update documentation.yaml files from v1 to v2 schema"
argument-hint: "Path to documentation.yaml file or directory"
---

Update the provided documentation.yaml file from v1 schema to v2 schema following these migration rules:

**MANDATORY REQUIREMENT**: The updated documentation must include a minimal docker run example for the image.

**Important**: When URLs (website, issues, source-code) are missing, prompt the user for manual input rather than attempting to auto-determine them.

## Schema Changes (v1 → v2)

**Version Update**: 
- Change `version: 1` to `version: 2`

**New Required Fields**:
- `website`: URL to original software's web pages
- `docker`: Minimal docker run example structure (required)

**New Optional Fields**:
- `issues`: URL for submitting issues (defaults to https://bugs.launchpad.net/ubuntu-docker-images if not provided)  
- `source-code`: URL to source code repository
- `config`: Runtime configurations (replaces v1 `parameters`) - Dict[str, Dict[str, str]]
  - Each config option needs: `type` (required), `description` (required), `default` (optional)
  - Config type must be one of: "env", "mount", "port", "app", "other"
    - **env**: Environment variable
    - **mount**: Data mount (like a volume)
    - **port**: Port mapping
    - **app**: Application input/argument
    - **other**: Miscellaneous capability or setting

**Docker Structure** (required):
- `docker.run_cmd`: Command to run inside the container (use "--args <named-args>" if no additional command)
- `docker.parameters`: Additional docker run parameters (everything except the image name)
- `docker.run_conclusion`: Expected outcome with specific examples (e.g. "You'll see 'Foo Bar' printed to STDOUT", "Access the web application at localhost:8080") (required if no dockerfile)
- `docker.dockerfile`: Example Dockerfile (required if no run_conclusion)
- `docker.legacy`: For mixed Docker/rock repositories

**Obsolete Fields to Remove/Transform**:
- `parameters`: Complex parameter list → migrate to `config` structure with proper type classification
- `debug`: Debugging information → remove, content goes to `docker` section
- `docker.access`: Remove if present  
- `is_chiselled`: Remove if present

**Structure Requirements**:
- `docker` field is required
- Inside `docker`, order fields as: `run_cmd`, `parameters`, then `run_conclusion`/`dockerfile`/`legacy` as applicable
- At least one of `docker.run_conclusion` OR `docker.dockerfile` must be provided
- If using `docker.legacy.*`, then `docker.legacy.run_conclusion` is required
- Config types must be one of: "env", "mount", "port", "app", "other"
- In `config`, list options grouped in this order: `env`, `mount`, `port`, `app`, `other`
- Config key format:
  - For `env` entries, use the environment variable name as the key (example: `TZ`)
  - For `mount`/`port`/`app`/`other`, use the quoted CLI-style key (example: '`-v <path>:/alertmanager`')
- The `description` field must start with: `application_name is ...`
- Do not insert a blank line between `description` and `website`
- Insert a blank line between the end of the `docker` block and `config`
- Top-level field order must be:
  1. `version`
  2. `application`
  3. `description`
  4. `website`
  5. `issues`
  6. `source-code`
  7. `docker` (required)
  8. `config` (if present)

## Migration Process

1. **Read the current v1 file**
2. **Update version number** to 2
3. **Add required fields**: `website` URL and `docker` structure
   - **Prompt for manual input** of URLs (website, issues, source-code) when missing
4. **Transform obsolete fields**:
   - `parameters` → migrate to `config` structure by:
     - Classifying each parameter by its type (env, mount, port, app, or other)
     - Writing keys as `ENV_VAR_NAME` for `env`, and as quoted CLI-style keys for non-env types (for example, '`-v <path>:/alertmanager`', '`-p <port>:4434`')
     - Ordering `config` entries by type: env, mount, port, app, other
     - Extracting the description from the parameter documentation
     - Including default values where applicable
   - `debug.text` content → `docker` section components
   - Use only `config` structure for runtime configuration options (do NOT duplicate in `docker.parameters`)
5. **Remove obsolete fields**: `debug`, `docker.access`, `is_chiselled`
6. **Add optional fields** as appropriate (issues, source-code)
7. **Enforce top-level field order**: `version`, `application`, `description`, `website`, `issues`, `source-code`, then `docker`, then `config` (if present)
8. **Validate schema compliance** - ensure required docker structure and constraints

**Note**: This prompt processes **single files only**. For batch updates, run the prompt multiple times.

## Config Migration Example

**Config Structure in V2**:
```yaml
config:
  TZ:
    type: env
    description: Container timezone.
    default: 'UTC'
  '`-v <path>:/alertmanager`':
    type: mount
    description: Mount Alertmanager data directory.
  '`-p <port>:4434`':
    type: port
    description: Kratos admin API port.
```

## Example Transformation

**V1 Example**:
```yaml
version: 1
# --- OVERVIEW INFORMATION ---
application: karma
description: >
  Karma is an alert dashboard for Prometheus Alertmanager.
  Alertmanager UI is useful for browsing alerts and managing silences,
  but it’s lacking as a dashboard tool - karma aims to fill this gap.
  Read more on the [official website](https://karma-dashboard.io/).
# --- USAGE INFORMATION ---
docker:
  parameters:
    - -p 8080:8080
  access: Access your Karma instance at `http://localhost:8080`.
parameters:
  - type: -e
    value: 'TZ=UTC'
    description: Timezone.
  - type: -p
    value: '8080:8080'
    description: Expose Karma on `localhost:8080`.
debug:
  text: |
    ### Debugging
    
    To debug the container:

    ```bash
    docker logs -f karma-container
    ```

    To get an interactive shell:

    ```bash
    docker exec -it karma-container /bin/bash
    ```

**V2 Result**:
```yaml
application: karma
description: |
  Karma is an alert dashboard for Prometheus Alertmanager.
  Alertmanager UI is useful for browsing alerts and managing silences,
  but it is lacking as a dashboard tool - karma aims to fill this gap.
website: https://karma-dashboard.io/
issues: https://github.com/canonical/karma-rock
source-code: https://github.com/canonical/karma-rock

docker:
  run_cmd: --args <named-args>
  parameters: >-
    -e TZ=UTC
    -p 8080:8080
  run_conclusion: |
    Karma starts on port 8080.
    Access the dashboard at http://localhost:8080 after running with a port mapping, for example:
    docker run -e TZ=UTC -p 8080:8080 ubuntu/karma:latest

config:
  TZ:
    type: env
    description: Timezone.
    default: 'UTC'
  '`-p <port>:8080`':
    type: port
    description: Expose Karma on localhost:8080.
```