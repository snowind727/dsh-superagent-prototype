# SETUP

Instructions for an coding agent that sets up `dsh-superagent` for its operator. If you are a person, ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent for me.

## How to run the setup (Crucial Rules for the Agent)

The setup follows [docs/000-philosophy.md](docs/000-philosophy.md): **take as little of the operator's attention as possible.**

- **Default to Feishu-only / Local Workspace mode.** Unless the operator explicitly asks to set up GitHub repository automation, always set up `dsh-superagent` in **Feishu-only mode** as a personal daily session orchestrator.
- **Assume the operator already has a Feishu bot (`App ID` and `App Secret`).** Do **NOT** instruct the operator on how to create a Feishu app or configure Feishu permissions unless they ask.
- **Do NOT stop or wait in the middle of setup.** Do **NOT** ask questionnaires before installing, and do **NOT** pause halfway waiting for the operator to input their `App ID` or `App Secret`. Never ask for secrets in the chat.
- **Execute Steps 1 to 5 below end-to-end automatically.** Fix any missing environment prerequisites yourself. Only after the service is running (`runtime: "ready"`) and pre-configured, send **one final handover message** (Step 6) containing the exact terminal commands for the operator to bind their Feishu bot credentials on their own.

---

## Step 1. Probe and install prerequisites automatically

Run from this checkout:

```sh
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
node -v; pnpm -v; git --version; tmux -V; python3 --version; dsh --version; command -v lark-cli
```

Resolve any missing items yourself without stopping to ask:

1. **`dsh` CLI:** If `dsh` is not on `PATH`, locate the installed DeepSeek Harness binary (for example, on macOS at `"/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh"`) and symlink it:
   ```sh
   mkdir -p ~/.local/bin
   ln -sf "/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh" ~/.local/bin/dsh
   ```
2. **`tmux`:** If missing, install it automatically (e.g., `brew install tmux` on macOS or the system package manager on Linux).
3. **`lark-cli`:** If missing, install the official Feishu/Lark CLI globally:
   ```sh
   npm install -g @larksuite/cli
   ```

---

## Step 2. Install dependencies and build the bundle

From the repository root, install dependencies and build `lib/`:

```sh
pnpm install && pnpm build
```

*(If the operator explicitly requested running the full test suite, run `pnpm check`.)*

---

## Step 3. Create the `lark-bot` wrapper script with auto-binding

Create `~/.local/bin/lark-bot` so that:
- It isolates the bot's configuration in `$HOME/.lark-cli-dsh-superagent`.
- When the operator later runs `~/.local/bin/lark-bot config strict-mode bot --global` in their own terminal, the script automatically registers `~/.local/bin/lark-bot` with the running `dsh-superagent` Management API (`http://127.0.0.1:3082/orchestrator/api/settings/source:feishu`) and triggers a hot restart so Feishu connects immediately without needing another round-trip with the agent.

Run:

```sh
mkdir -p ~/.local/bin
cat > ~/.local/bin/lark-bot <<'EOF'
#!/bin/sh
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
if [ "$1" = "config" ] && [ "$2" = "strict-mode" ] && [ "$3" = "bot" ]; then
  lark-cli "$@" || exit $?
  api="http://127.0.0.1:${DSH_SUPERAGENT_PORT:-3082}/orchestrator/api"
  rev=$(curl -s -H 'x-orchestrator-request: 1' "$api/settings" 2>/dev/null | python3 -c "import sys,json; print(next(s['revision'] for s in json.load(sys.stdin)['sources'] if s['id']=='feishu'))" 2>/dev/null)
  if [ -n "$rev" ]; then
    curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' "$api/settings/source:feishu" \
      -d "{\"value\":{\"enabled\":true,\"config\":{\"cli\":\"$HOME/.local/bin/lark-bot\",\"chats\":[],\"directMessages\":true,\"otherChats\":\"mentions\",\"readPageSize\":20,\"linkTimeoutMs\":5000}},\"revision\":$rev}" >/dev/null 2>&1
    echo "Feishu bot CLI bound to dsh-superagent; service is restarting to connect."
  fi
  exit 0
fi
exec lark-cli "$@"
EOF
chmod +x ~/.local/bin/lark-bot
```

---

## Step 4. Create the `superagent` profile and sync model configuration

Follow [docs/006-deployment.md](docs/006-deployment.md) to set up the isolated `superagent` profile:

1. **Create the profile and link this bundle** (skip the first command if `~/.dsh/profiles/superagent` already exists):
   ```sh
   dsh superagent --from-default-profile web --dump-config > /dev/null
   dsh plugin --profile superagent add "link:$PWD"
   mkdir -p "$HOME/.dsh/state/superagent"
   ```
2. **Configure `~/.dsh/profiles/superagent/cordis.patch.yml`:**
   - Start with the required entries for `webserver` (port `3082`), `dsh-superagent` (`stateDir` set to the **absolute path** of `$HOME/.dsh/state/superagent`), `dsh-superagent-feishu` (`disabled: false`), and `dsh-superagent-github` (`disabled: true` in Feishu-only mode):
     ```yaml
     - id: webserver
       config:
         host: 127.0.0.1
         port: 3082
     - id: dsh-superagent
       config:
         stateDir: <absolute path to $HOME/.dsh/state/superagent>
     - id: dsh-superagent-feishu
       disabled: false
     - id: dsh-superagent-github
       disabled: true
     ```
   - **Crucial:** Inspect the operator's existing `~/.dsh/profiles/desktop/cordis.patch.yml` (or `~/.dsh/profiles/web/cordis.patch.yml`) and append their existing entries (such as `llm-deepseek`, `llm-pi-ai`, `agent-default-model`, `permission`, `web`, etc.) into `~/.dsh/profiles/superagent/cordis.patch.yml` so the `superagent` profile has working model providers out of the box.

---

## Step 5. Start the service and pre-configure Policy & Prompts

1. **Start the background service:**
   ```sh
   scripts/dsh-superagent start
   scripts/dsh-superagent status
   ```
   Verify that `scripts/dsh-superagent status` exits `0` and `health.runtime` is `"ready"`.

2. **Pre-configure Feishu Policy and Local Daily Session Prompts via the Management API (`http://127.0.0.1:3082/orchestrator/api`):**
   Run this script to enable direct messages and `@bot` mentions in Feishu (so any message to the bot in DM or `@bot` in groups is admitted as an explicit request without needing to look up `open_id`s first) and tailor the prompts for local daily session management:

   ```sh
   python3 - <<'PY'
   import json, urllib.request

   api = "http://127.0.0.1:3082/orchestrator/api"
   headers = {"x-orchestrator-request": "1", "content-type": "application/json"}

   def get(path):
       with urllib.request.urlopen(urllib.request.Request(f"{api}/{path}", headers=headers)) as r:
           return json.load(r)

   def put(path, payload):
       req = urllib.request.Request(f"{api}/{path}", data=json.dumps(payload).encode(), headers=headers, method="PUT")
       with urllib.request.urlopen(req) as r:
           return json.load(r)

   settings = get("settings")
   policy_rev = settings["blocks"]["policy"]["revision"]
   policy_val = settings["blocks"]["policy"]["value"]
   policy_val["sources"]["feishu"] = {
       "allowed": [],
       "syncChat": "",
       "admission": {
           "direct": {"unlistedAddressed": "explicit"},
           "other": {"unlistedAddressed": "explicit"}
       }
   }
   put("settings/policy", {"value": policy_val, "revision": policy_rev})

   prompts = get("prompts")
   base_rev = prompts["current"]["revision"]
   cfg = prompts["current"]["config"]
   cfg["communication"] = (
       "Return to the source Feishu conversation to deliver results or ask necessary questions. "
       "Reply in Simplified Chinese (or the language of the user's message), keeping each Feishu message concise (under 100 words, enforced by the transport). "
       "When a worker reports a ready outcome (or answers a question), deliver the key conclusion, changed files/paths, or answer directly in the originating Feishu topic thread. "
       "A GitHub pull request is NOT required unless the user explicitly asks for one."
   )
   cfg["scope"] = (
       "Handle coding tasks, codebase exploration, debugging, local scripts, shell operations, and technical questions requested by the user. "
       "Workers may inspect and modify any local directory the user specifies (such as ~/Documents/code/...) or work inside their own ticket workspace directory ($DSH_SUPERAGENT_DIR). "
       "Pause and ask a concrete human_action question only when a critical decision or missing prerequisite blocks progress."
   )
   cfg["delivery"] = (
       "Local tasks, answers, and code changes do NOT require opening a GitHub PR unless the user explicitly asks for a PR. "
       "When the requested work or analysis is complete and verified locally, call orchestrator_report with state='ready' (omit pr_number) and a clear summary so the orchestrator can deliver the result to the user in Feishu."
   )
   put("prompts", {"config": cfg, "baseRevision": base_rev})
   PY
   ```

---

## Step 6. Hand over to the operator

Once Steps 1–5 are complete and `scripts/dsh-superagent status` exits `0`, send **one concise message** to the operator containing:

1. **Service & Panel URL:** Read the newest `.artifacts/dsh-superagent-superagent-*.log` to show the Web UI URL (`http://127.0.0.1:3082/?token=...`), and note that **Superagent** in the left sidebar opens the management panel.
2. **Exact commands for the operator to bind their Feishu bot in their own terminal** (replacing `<App_ID>` with their `cli_...` App ID, then pasting their `App Secret` when prompted):
   ```sh
   ~/.local/bin/lark-bot config init --app-id <App_ID> --app-secret-stdin
   ~/.local/bin/lark-bot config strict-mode bot --global
   ```
   Explain that running the second command automatically binds `lark-bot` to the running `dsh-superagent` service and connects the Feishu WebSocket consumer (they can verify with `scripts/dsh-superagent status` that `health.feishu` becomes `"ready"`).
3. **How to use and manage the service:**
   - Send a direct message to the Feishu bot (or `@bot` in a group) to start a task; reply inside the Feishu topic thread to follow up on that same task session.
   - Start / stop / restart / check status from the repository root with `scripts/dsh-superagent start | stop | restart | status` (see [docs/001-domain.md](docs/001-domain.md) and [docs/006-deployment.md](docs/006-deployment.md) for details).
