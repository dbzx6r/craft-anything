# Using Codex with Craft Anything

Craft Anything reaches Codex in two ways, and both have to get past Codex's own tool approval.

## Use your ChatGPT sign-in on macOS

Settings → AI → Found on this computer → Harnesses offers **Codex**. Add it
and use **Check** in its configured row to check the executable and its sign-in. The check calls only
`--version` and `login status`; it sends no prompt and does not prove that a model
or quota is available. Send a message to verify those separately.

An app opened from the Dock does not inherit a terminal's PATH. Codex detection
and execution both search the known CLI locations inside ChatGPT.app and
Codex.app, in the user's Applications folder and then the system Applications
folder, after trying PATH. A program selected explicitly in Settings stays first.
Only executable files are accepted; the desktop app itself is never launched as
a command-line agent. No shell profile or symlink is needed.

Codex manages the login. To connect it, run the selected Codex executable with
`login` in a terminal and choose **Sign in with ChatGPT**. Craft Anything never
reads, copies or stores the Codex credentials. This is distinct from the OpenAI
API-key backend: no API key is needed to use a Codex ChatGPT login. A known
signed-out result stops the connection test/build with sign-in guidance; an
unrecognised result is reported as unverified, never as connected.

### Fork verification (2026-11-09)

- Rust 1.99.1 on Apple Silicon: the desktop app and companion CLI build with the
  locked dependencies. This local fork build uses the optimized development
  profile, not a notarized upstream release.
- AI library: 558 passed, 3 existing ignored. GUI library: 94 passed, 1 existing
  ignored. The checks cover bundle lookup and executable permissions, root
  precedence, signed-out and unknown status, authentication guidance and the
  connection result in Settings. Ignored tests were not counted as verified.
- Clippy with warnings denied for both changed crates and all their targets,
  formatting, layering, lock discipline, translation coverage and secret checks
  pass. No new dependency or CI change.
- With PATH limited to the usual Dock directories and only the test app's own
  configuration isolated, discovery found the bundled Codex and its existing
  ChatGPT sign-in. A real chat returned the requested marker. A disposable static
  counter project completed Study → Build → Check → Ready through the actual
  Codex/MCP integration; its Node test also passed independently.
- The installed bundle passed strict/deep code-sign verification with a local
  ad-hoc signature. The native Settings Check button displayed signed-in status.
  No approval bypass flags, credential copying, global Codex config changes or
  user-project changes were needed.

Existing limitations remain: native ChatGPT OAuth inside Craft Anything is not
added here; Codex owns authentication. Session resume/model-picker work is
separate (upstream PRs #633/#632), and this verification is not a Windows/Linux
live-account test. A login-status check alone never proves model entitlement or
remaining quota. An upstream/launcher update can replace a locally installed
fork; keep its source and original-app/settings rollback copy before updating.

## Why a tool can be refused

`codex exec` is non-interactive: it has no one to ask, so its approval policy is `never`. A tool
that an MCP server doesn't mark read-only (such as `rebuild_start`, which runs a coding agent) is
refused before our server sees the call, with `requires approval, but approval policy is never`.
Codex's own settings decide this; Craft Anything's consent is a second, separate check.

## What we set

Only tools that ask the person themselves are approved on Codex's side
(`AGENT_APP_PREAPPROVED_TOOLS` in the engine, today just `rebuild_start`; a test checks each one
asks for consent). Every other tool keeps whatever approval policy you set in Codex.

- **Connecting Codex to Craft Anything** (`craft-anything-cli setup --client codex --yes`, or the
  Connect screen) writes the entry in `~/.codex/config.toml` (`$CODEX_HOME/config.toml` if set):

  ```toml
  [mcp_servers.craft-anything]
  command = "…/craft-anything-cli"
  args = ["mcp"]
  startup_timeout_sec = 30
  tools = { rebuild_start = { approval_mode = "approve" } }
  ```

  Your other entries and comments stay as they are, the old file is kept as
  `config.toml.craft-anything.backup`, and settings you added to our entry (`env`,
  `enabled_tools`, approvals for other tools) are kept when setup updates it. Setting
  `approval_mode` for `rebuild_start` yourself is overwritten by the next setup run.
  `craft-anything-cli doctor --client codex` reports an entry without the approval as "a different
  copy or shape of the command"; running setup again fixes it.
- **Building with Codex from the app** starts `codex exec` with the same setting on its `-c`
  flags, so nothing in your config is needed. Chats and builds also start the Craft Anything MCP
  server with `--all-tools`: Codex doesn't refresh its callable tool list after `enable_toolset`,
  so every tool permitted by the current mode is listed from startup. Clean room restrictions still apply.

## Who still asks

Approving `rebuild_start` in Codex doesn't remove Craft Anything's own consent: it runs a coding
agent, so our server asks before it runs. Who can answer depends on how the server was started:

- **Builds started from the app** run the server with the project's inbox, so the person answers
  in the app (Allow / Don't allow).
- **The entry from `setup`** starts `craft-anything-cli mcp` on its own, with no app attached, so
  there is no approval card: the call returns `consent_required` and nothing runs. To let Codex
  start builds this way, add `"--allow-run"` to the entry's `args` (every tool that runs code is
  then treated as allowed, so only do it for a Codex you trust), or start the build from the app.

In Clean room mode the tools that touch the original stay refused either way.

## If it still fails

A refusal for one of our tools now says what was blocked and how to fix it, instead of showing
`null`; another MCP server's refusal keeps Codex's own words, because our setup can't change
that server's policy. To approve more of our tools by hand, add
`[mcp_servers.craft-anything.tools.<tool>]` with `approval_mode = "approve"` and start the chat
again. Approving all of them (`default_tools_approval_mode = "approve"`) also approves tools with
no consent check of their own, such as `project_inbox_answer`, which can answer a pending approval,
so we don't do it.

Startup tool discovery was verified with Codex CLI 0.166.0, a loopback model and the real MCP server:
Codex received 107 Clean-room-permitted tools, including non-Core tools, while Deep-only tools stayed
hidden. No real-account build was run.
