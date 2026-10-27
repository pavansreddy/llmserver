# llmserver Trust — design for model and skill supply-chain security

Status: **current to 2.7.3.** The table lists every release; the implementation notes below it say what was built and how it differs from the original design (sections 1 to 15, kept as written for 1.7.0).

| Release | Contents |
|---|---|
| 1.7.0 | Models |
| 1.8.0 | Skills and `llmserver hf` |
| 1.9.0 | Air-gap transfer, SBOM, local scanners |
| 2.0.0 | Sandbox and `eval` |
| 2.1.0 | One folder, migration, movable home |
| 2.1.1 | Clearer messages: missing repo vs gated, name suggestions |
| 2.2.0 | No hand-editing: `model`, `config`, `key`, `autostart` commands |
| 2.3.0 | `llmserver chat` |
| 2.3.1 | Fixes from first use: bench on reasoning models, chat prompt, GGUF hints |
| 2.4.0 | Local GGUF files, your own engines by path, install from a downloaded release file |
| 2.4.1 | chat speed line counts thinking tokens from the first token of any kind |
| 2.4.2 | chat `--reasoning`; bench uses the server's real context; Ollama CPU offload shown in bench and ps |
| 2.5.0 | Detailed bench: prefill/decode, TTFT, TPOT, ITL percentiles, E2E, thinking vs answer, `--json` |
| 2.6.0 | The gateway, and Open WebUI as a managed, sandboxed add-on |
| 2.7.0 | Hybrid RAG with a pinned reranker; web search through a private, pinned SearXNG |
| 2.7.1 | `webui_reranker best`: bge-reranker-v2-m3 on llama.cpp through the gateway's `/v1/rerank` |
| 2.7.2 | Review fixes: sandbox gets its own PID namespace and no session sockets; clean engine environment; trusted system settings file; Open WebUI's first account on loopback; `AUTH=key` everywhere; time+size check with rehash; gateway on demand, limits and `/metrics`; HTTPS for Open WebUI with your certificate; `backup`/`restore`; `models prune` |
| 2.7.3 | `models share on\|off`: other tools use the private model store through `HF_HUB_CACHE` and `OLLAMA_MODELS` in the login files; `uninstall --purge-models` keeps other tools' files in `models/` |

### Implementation notes (2.7.3): sharing the private model store

The request was to move `~/.cache`, `~/.config`, `~/.local`, `~/.ollama`, `~/.agents` and `~/.claude` into the home and link them back. Rejected: those folders hold every program's settings, keys and caches. In the home they would fall under `uninstall`, `restore` and the "llmserver owns everything here" rule, and one wrong bind would show them to a sandbox. llmserver already links only what it owns (one link per skill). What other tools actually need is the model files, and two variables give exactly that.

- **Mechanism:** a block between `# >>> llmserver models share >>>` and `# <<< llmserver models share <<<` in `~/.profile`, in `~/.bash_profile` or `~/.bash_login` when bash reads that file instead (skipped when it sources `~/.profile`), and in `~/.zshenv` for zsh. Plain POSIX `export` lines with single-quoted paths; a home path with a quote is refused. No file in the home is sourced by a login shell, so a write to the home cannot become code in every login.
- **Safe when stale:** each line is guarded by `[ -d PATH ]`, so after an uninstall or a move it does nothing. `uninstall` removes the block; `doctor` warns when the block points outside the current home or the store is `shared`.
- **Careful edits:** written to a temporary file in the same folder and renamed, with the old mode; through a link to its target (dotfile managers); unchanged files are not rewritten; an unfinished block (begin without end) is kept as it is, so no line that may be the user's is dropped.
- **User values win:** a variable the user set (shell or settings file) is not written, as llmserver itself already gives it priority.
- **Only with `MODEL_STORE=private`:** with `shared`, llmserver and other tools already use the standard caches; `on` says so and changes nothing.
- **Integrity:** other tools can now change files in `models/`. The time+size check with rehash (2.7.2) catches it at the next start. `uninstall --purge-models` no longer deletes the whole `models/` folder: it deletes the recorded downloads (as with the shared caches) and empty folders, and keeps other tools' files.
- **Not done (KISS):** links, `environment.d` (no conditions, so a stale path would re-create folders), systemd unit overrides, fish. The manual says how to set the variable for a service.

### Implementation notes (2.7.2): fixes from the review

Principle kept: one script, no Docker, no sudo; every download pinned and checked; safe by default. Each fix is the smallest change that closes the gap.

- **Sandbox escape through `/proc` closed:** without `--unshare-pid` the engine saw every process of the user and could open `/proc/PID/root/...` (the outer filesystem, home included) and `/proc/PID/environ`. All three sandboxes (engines, Open WebUI, SearXNG) now come from one `sandbox_base` with `--unshare-pid`. `sandbox_works` tests that flag too. Stopping still signals the process group: bwrap stays the group leader in the host namespace, and the server inside keeps that group.
- **Session sockets hidden:** a read-only bind does not stop `connect()` on a Unix socket, so `/run/user/UID` (and `XDG_RUNTIME_DIR` when it is elsewhere) is a tmpfs: no D-Bus session bus (and so no `systemctl --user` escape), no SSH agent, no keyring. Still reachable, and stated: abstract sockets (shared network namespace) and the system D-Bus. bwrap is called by its full path (the services start with a short `PATH`).
- **Clean engine environment:** engines inherited the whole shell environment (cloud keys, tokens, `SSH_AUTH_SOCK`). Now an allowlist (`ENGINE_ENV_RE`: `PATH`, `HOME`, locale, `LD_LIBRARY_PATH`, and the prefixes of GPU and engine programs) plus every variable of the settings file. With or without the sandbox. The engine's own API key stays in its environment (stated as a limit).
- **System settings file:** `/etc/llmserver_env` can name programs (`*_SOURCE`) and set `LD_PRELOAD`. It was read regardless of its owner. Now it is used only when owned by root or the user, not writable by group or others, and in a folder that others cannot change (or a sticky one). `SECURITY_FLOOR` keeps its stricter root-only rule.
- **First Open WebUI account:** the first account becomes the admin, and the default host is `0.0.0.0`. While the user table is empty (or cannot be read), `webui start` binds `127.0.0.1` and says how to create the account (locally or through `ssh -L`); `webui restart` then opens it to the network. No dependency on Open WebUI internals beyond the `user` table that `status` already read.
- **`AUTH=key` means every client:** the gateway accepted loopback clients without a key. Now with `AUTH=key` it needs the key from everyone (constant-time comparison); Open WebUI gets the key as its connection key (`OPENAI_API_KEY`, and `openai.api_keys[0]` in its database through the same "only when llmserver's value changed" rule; other connections the admin added are kept) and for the external reranker. `doctor` warns when other accounts can log in and `AUTH` is not `key`.
- **Integrity at start:** records now keep each file's modification time. A start compares size and time; a new time with the same size rehashes that file and passes (and updates the record) only when the hash matches. This catches a same-size swap at the next start, not only at `trust verify`. Import refreshes the times of the placed files; local GGUF files use the same rule (a `touch` no longer asks again).
- **`--yes` no longer travels:** `chat MODEL --yes` and the best-reranker start called the start with `--yes`, which also accepted security warnings. Both now start without it: a model with open warnings needs the user's own start.
- **Script checksum:** `doctor` prints the SHA-256 of the running script; the manual tells the user to check it against the release's `SHA256SUMS`. (Signed releases and self-update stay out of this line, by choice.)
- **Honest locks:** the manual now says that the vLLM, Open WebUI, SearXNG and garak locks are made at the first install (trust on first use, then fixed).
- **Functional fixes:** `upgrade` without a component updates only what is installed (it installed every engine); `webui stop` stops only the services `webui start` started (a gateway the user started stays); a bare `llama:REPO` means Q4_K_M in `check` as in the security check (the size lookup took the smallest quant), with a message that lists the quants; vLLM also gets small text files in subfolders (`1_Pooling/`, processors); looking at an Ollama model (`ps`, `capacity`) gives its log its old time back, so it does not reset the idle time; MIG GPUs give a warning; a vLLM start that fails for KV-cache memory says so; llmserver calls itself as `"$BASH" "$SELF"` (works without execute permission); `config set` names the other setting when that one is broken; menu flags accept '...' and "..." for values with spaces (`split_words`/`join_words`; no `eval`).
- **Gateway:** optional models on demand (`GATEWAY_AUTOLOAD`, default off): it lists the menu models that have a check record (a file written by llmserver whenever the menu or the records change) and starts one with llmserver itself (`_LLMSERVER_NO_FETCH=1`: never a download, never a question; eviction as `EVICT`, `ask` acts as `never`); `GATEWAY_IDLE_STOP` minutes stops only what it started. One start at a time. Its environment is the engine allowlist plus `HOME`, `LLMSERVER_HOME` and the proxy variables. Also: 64 concurrent requests (then 503), a 60 s idle timeout for clients, `Content-Length` checks, and `/metrics` (requests, in flight, running, started by the gateway; no message content).
- **HTTPS for Open WebUI:** with `WEBUI_TLS_CERT` and `WEBUI_TLS_KEY`, Open WebUI runs as `python -m uvicorn open_webui.main:app --ssl-certfile --ssl-keyfile --forwarded-allow-ips 127.0.0.1` (the same app `open-webui serve` starts; `FROM_INIT_PY=true` as `serve` sets it). Your certificate only: no local CA (keeps it small). The two files are bound read-only into the sandbox.
- **`backup`/`restore`:** one tar (mode 600) of what cannot be downloaded again: settings, `config/`, `trust/records`, ports, pins, skills, Open WebUI data (its SQLite database through SQLite's backup API). `restore` refuses a file with any member outside that list, any link or device, and works only while models and services are stopped.
- **`models prune`:** deletes the files of older commits that llmserver downloaded (from `state/downloads`), keeping every path that a current record or a running model's record uses (`trustpy keep-paths`), and blobs that another snapshot links to (the uninstall logic). Older records go too.
- **Speed:** `list`, `check` and `capacity` check every menu record in one Python start (`enforce-many`), instead of one per entry.
- **Not done, on purpose (KISS):** several copies of one model, per-user keys and rate limits, a local certificate authority, self-update and signed releases, metrics beyond counters, MCP tools.
- **Tests:** real runtime install, real Hugging Face (vLLM file selection, default quant message, no-fetch refusal), a stand-in llama.cpp and Open WebUI, and a stand-in bubblewrap that records its arguments (bubblewrap was not available): engine environment, sandbox arguments, on-demand start and idle stop, `AUTH=key` on the gateway, first Open WebUI start on loopback and the restart to the network, a touched and a changed model file, quoted menu flags, backup and restore (and a refused file), prune with two simulated commits. `bash -n`, shellcheck (warning level, clean) and a parse of every embedded Python program.

### Implementation notes (2.7.0 and 2.7.1): documents and web search

- **Hybrid RAG is Open WebUI's own:** BM25 (`rank_bm25`) and vector search, merged (`EnsembleRetriever`, weight 0.5), then `RerankCompressor` with the reranker. llmserver sets it up: 20 candidates from each search, best 6 kept (`rag.top_k`, `rag.top_k_reranker`). No extra retrieval code in llmserver.
- **Fast reranker:** cross-encoder/ms-marco-MiniLM-L6-v2 at a pinned commit, the six files the CrossEncoder loads, each against a pinned SHA-256, fetched like the embedding model. Measured on two CPU cores for 20 passages: 1.4 s (MiniLM), 7.3 s (mxbai-rerank-xsmall), 29 s (bge-reranker-v2-m3): the best one is not usable on a CPU in-process, hence 2.7.1.
- **Security gap closed:** Open WebUI defaults `RAG_EMBEDDING_MODEL_TRUST_REMOTE_CODE` and `RAG_RERANKING_MODEL_TRUST_REMOTE_CODE` to `True`; llmserver sets both to `False` (and the reranker's auto-update off). Also `FORWARDED_ALLOW_IPS=127.0.0.1` (Open WebUI trusts `X-Forwarded-For` from anyone by default, and it faces the network directly), `ENABLE_LOCAL_WEB_FETCH=False` explicitly, and `STATIC_DIR` in its data folder (it tried to write into its read-only venv at every start).
- **Settings in its database:** Open WebUI 0.12.3 stores config per key in table `config`; environment values only seed missing keys. Before each start (Open WebUI stopped), llmserver writes a managed key only if its value differs from what llmserver wrote last (`webui/llmserver-applied.json`): an admin's change survives restarts until the matching llmserver setting changes. The same values go in the environment for the first start.
- **SearXNG:** no releases, so a pinned commit plus `SEARXNG_TREE`, a SHA-256 over `searx/**` and the requirement files, computed after unpacking: the same check whether the source came from GitHub's archive or (archive host unreachable) `git fetch --depth 1` of the commit. Only `searx/`, the two requirement files and LICENSE are kept; `version_frozen.py` is written so it never calls git. Packages hash-locked (`uv pip compile --generate-hashes` of its pinned requirements, `pip sync --require-hashes`), Python 3.12, 125 MB. Runs granian (WSGI, 1 worker) on `127.0.0.1` only, in bubblewrap, secret in `config/searxng-secret` (600), settings generated each start (limiter off, JSON on, metrics and image proxy off) unless `config/searxng.yml` exists. Off by default and asked at setup: questions leave the computer.
- **Proxies:** both services get the proxy and CA variables, with `127.0.0.1,localhost,::1` added to `NO_PROXY` (Open WebUI's calls to the gateway, the reranker and SearXNG must never go through a proxy); CA files named in the environment are bound into the sandbox (its home is empty).
- **Best reranker (2.7.1):** `llama:gpustack/bge-reranker-v2-m3-GGUF:Q8_0@3093af0…` started by `webui start` as a normal llmserver model (security check, VRAM placement, sandbox) with `--ctx 8192 -- --reranking -b 2048 -ub 2048`, not added to the menu or as the last model. Its record has `ROLE=rerank`; the gateway hides such models from `/v1/models` and routes `/v1/rerank` (also `/rerank`, `/v1/reranking`) only to them. Open WebUI uses `rag.reranking_engine=external` with the gateway URL. It falls back to the fast reranker when llama.cpp is not enabled or the start fails, and `webui stop` stops it only if `webui start` started it.
- **Tests:** 52 cases (`rag_suite`) with the real Open WebUI: settings, pinned-source refusal, git fallback, environment, document question answered first by hybrid search, admin change kept, llmserver change applied, the best reranker through the gateway (stub llama.cpp), fallback, SearXNG in the sandbox and not on the network, real searches, busy port, uninstall. Plus a real run: real bge-reranker-v2-m3 on real llama.cpp ranked a French passage first for an English question (1.9 s on two CPU cores for four passages).

### Implementation notes (2.6.0): the gateway and Open WebUI

- **Why a gateway:** each model has its own port, and Open WebUI stores its connection list in its database at first launch (env changes are ignored afterwards; verified). The gateway (`llmserver gateway`, stdlib Python on llmserver's runtime, default `127.0.0.1:8000`) lists the running models from llmserver's records on every request and forwards by the `model` field, streaming (`/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/responses` only; 64 MB body limit). Loopback clients never need a key; when bound to the network, others need the API key, and it refuses to start on the network with `AUTH=none`. Its log has no message content.
- **Why Open WebUI, not AnythingLLM:** compared on install, design and security (2026-10-22): Open WebUI installs with pip/uv (fits llmserver: pinned, in its own folder, no Docker, no sudo) and has accounts with an admin; AnythingLLM's server is Docker-only (bare metal "not supported"), single-user mode has no password by default, and PostHog telemetry is on by default.
- **Install:** `install webui`: Open WebUI 0.12.3 on Python 3.12.15 (it needs 3.11/3.12), CPU PyTorch, hash-locked (`uv pip compile --generate-hashes`, then `pip sync --require-hashes`), relocatable venv in `tools/open-webui/<ver>` (2.4 GB). Its embedding model (all-MiniLM-L6-v2) is fetched once at a pinned commit, only the ten files sentence-transformers loads, each against a pinned SHA-256 (Open WebUI's own first-start download also takes pickle, ONNX, TF and other copies). `SENTENCE_TRANSFORMERS_HOME` (re-read at each start, so a moved home still works) points at that cache.
- **Run:** clean environment (`env -i`; no user tokens), `OFFLINE_MODE`/`HF_HUB_OFFLINE` (no network calls to Hugging Face at run time; verified), only the gateway as its connection (Ollama API off), community sharing and version check off, a generated `WEBUI_SECRET_KEY` (600), `DEFAULT_USER_ROLE=pending`. Open WebUI itself makes the first account the admin and closes sign-ups afterwards. Bound to `0.0.0.0:3000` by default (network use is intended); `WEBUI_AUTH=off` only on loopback. bubblewrap: read-only system, empty home and llmserver home, its venv and Python read-only, its data and home folders writable.
- **Integration:** service records in `state/services/`; `ps` shows the services; `autostart on webui` (a oneshot user unit); `uninstall` stops both and keeps `webui/` with `--keep-config`; `versions` lists it.
- **Tests:** 42 cases with the real Open WebUI (account creation, model list via the gateway, a model appearing without reconfiguration, chat, network login, environment, sandbox, no Hugging Face calls), and the gateway with and without the key from the network.

### Implementation notes (2.4.0): files on this computer

- **Local GGUF models** (`llama:/abs/path.gguf`; `~/`, `./` and relative paths are made absolute) go through the same gate with `source: file`: plan (parts of a split model, a matching `mmproj` in the folder), post (GGUF header, not a program, SHA-256, local scanners), record (`<id>@local.json`), and enforcement at every start (size and modification time; a change re-runs the checks). New rule `M-LOCAL-FILE` (ask at both levels, `--accept-risk`); strict still needs `trust approve`, as for every model. No Hub scan results exist for these files: this is stated, and the check records say so.
- The file is never llmserver's: it is not recorded as a download, so `model purge`, `--purge` and `uninstall --purge-models` never delete it; `export` refuses it. The sandbox binds its folder read-only. Clients use the file name (without `.gguf`) as the model id.
- **Engine paths:** `LLMSERVER_<ENGINE>_SOURCE` may be the program or its folder (also `bin/`, `build/bin/`). It counts as a system engine (never changed or deleted). The sandbox binds both the folder a symlinked program is called from and the folder it points to.
- **Release files:** `install --from FILE` recognises the pinned llama.cpp and Ollama release file names and installs them as llmserver's own copy after the pinned SHA-256 check (`LOCAL_SRC`: used before the network; everything else still downloads). Another release is refused with the way to use it as your own engine.
- **M-SIZE:** the Hub describes one GGUF per repository. When a repository holds GGUFs of more than one model (by file-name stem), that parameter count is no ceiling for the selected file, so the check is skipped (it caused a false warning on EldanRing/Winnow-12B).

### Implementation notes (2.3.1): fixes from first use

- **bench** counted only `text` tokens; Ollama's OpenAI endpoint streams a thinking model's tokens as `reasoning`, so no token was seen and the run failed with a bare "ValueError". It now counts text, content and reasoning, falls back to `/v1/chat/completions` when plain completions are missing or empty, and reports the reason with the error.
- **chat** prompt: llmserver's Python (python-build-standalone) uses libedit, which prints colour codes in an `input()` prompt literally; the prompt is plain now. Reasoning is hidden behind a live `thinking...` note and a "thought for Ns" line; `/think` or `--think` shows it.
- **vLLM on a GGUF repository** (`M-NO-WEIGHTS`) now says it is a GGUF repository and gives the llama.cpp command. When a repository has several GGUFs of the same quant, the file named after the repository is preferred.

### Implementation notes (2.3.0): chat

- `llmserver chat [MODEL] [MESSAGE]` talks to a running model through its own OpenAI-compatible endpoint (`/v1/chat/completions`, streamed), with the same API key handling as `env` and `eval`. It uses llmserver's own Python (standard library only); the client code is passed on a separate file descriptor so the user's stdin stays free.
- It never starts a model silently: in a terminal it asks; without one it needs `--yes`. Starting goes through the normal start path (security gate, VRAM guard, sandbox).
- One-shot mode (a MESSAGE argument or a pipe) writes only the answer to stdout, so scripts and agents can use it.
- Only the typed input lines are kept (`state/chat_history`, mode 600, for line editing); answers are not stored unless the user runs `/save`.

### Implementation notes (2.2.0): commands instead of hand-edited files

- **Menu:** `model list|add|rm|purge|edit`. `add` runs the gate's plan step to make sure the name exists (with "did you mean"); `setup` now does the same, which removes the way a mistyped name got into the menu. `rm` asks which entries in a terminal (numbers, ranges, names) and rewrites only those lines. `--purge` / `model purge` delete only files recorded in `state/downloads` for that model and not used by another menu entry (blobs shared with other snapshots are kept, as in uninstall), plus the model's check records; never for a running model, and only with a terminal answer or `--yes`.
- **Settings:** `config list|get|set|unset|edit`. Names are accepted short and lowercase (`port`); an unknown lowercase name is refused with the nearest setting, and engine variables must be in capitals. A new value is checked by running llmserver itself with the value in its environment (the environment wins over the file), so the same validation applies and nothing is written on failure. The file is edited in place (comments kept), stays mode 600, and secrets are masked in listings. `config` works even when the file is invalid, so it can repair it; `LLMSERVER_HOME` and administrator-only keys are refused.
- **Key:** `key show` (masked) and `key new` (asks, lists running models that still hold the old key).
- **Autostart:** `autostart on|off|list` writes `llmserver@.service` with the absolute script path and `LLMSERVER_HOME` when set; instances are named by the systemd-escaped spec, not the menu number. The unit file is removed when no instance is left; `uninstall` already removes it.
- **Command typos:** a bare word within a small edit distance of a command (and not an Ollama name in the menu) is reported as a mistyped command instead of being looked up as a model.
- **Messages** that told the user to set `LLMSERVER_X=...` or edit a file now give the `config set` / `model` command.

### Implementation notes (2.1.1): messages

- **Missing vs gated:** the Hub shows a gated model's details to anyone, so HTTP 401/404 from the revision API means "no such repository" (or private and not visible). That is now `M-NOT-FOUND`, with up to three close names from the Hub (`difflib` over the owner's models and a name search). `M-ACCESS` is kept for HTTP 403 or `X-Error-Code: GatedRepo`.
- **Availability is not a security verdict:** `M-NOT-FOUND`, `M-ACCESS`, `M-NO-WEIGHTS` and a blocking `M-HUB-OFFLINE` print one plain error ending in `[ID]`, not "refused by the security checks". Real rule failures print `security check failed: ... [ID]` and name the exact `--accept-risk` flags.
- **Names:** `:model` means the default engine; a mistyped engine (`vlm:...`) is named; a bad name shows the expected form. A mistyped menu entry points at `config/models`.
- **Network errors** are given in plain words (connection refused, name not resolved, timed out), not as Python exceptions. An unknown size is an info line ("memory check skipped"), not a warning.

### Implementation notes (2.1.0): one folder

- **Layout:** everything llmserver writes is in `~/.llmserver` (`LLMSERVER_HOME`, mode 700): `llmserver.env`, `config/` (menu, api-key, tuning), `logs/` (install.log, `models/`, ollama-pull.log, garak.log), `state/`, `skills/`, `trust/`, and the software (`engines/ runtime/ python/ uv/ bin/ tools/`). Model files stay in the shared caches unless `LLMSERVER_MODEL_STORE=private` (then `models/` in the home). No files in `/tmp`; locks stay in `state/` (cron has no `XDG_RUNTIME_DIR`).
- **Home location:** read only from the environment or `/etc/llmserver_env`. `LLMSERVER_HOME` inside `llmserver.env` is ignored with a warning (the file is inside the home). The old per-path overrides (`CONFIG_DIR`, `STATE_DIR`, `MODELS`, `LOG_FILE`, `ENV_FILE`) are gone.
- **Migration:** one time, automatic, from the XDG layout of 2.0 and older. It is blocked (old paths used for that run, with a warning) while a model started from the old layout is alive. Running-model records are re-pointed at `logs/models/`.
- **Moving the home:** `.llmserver-path` records the last location; on a change, symlinks under the home, `pyvenv.cfg` files, skill records, running records and the agents' skill links are rewritten from the old prefix to the new one.
- **Sandbox:** the home is a tmpfs inside bwrap; only `engines/` and `python/` are bound back, read-only. The engine never sees `llmserver.env`, the API key, logs, state or trust records (tested with the probe stub).
- **Skills:** the real copy is in `$LLMSERVER_HOME/skills/<name>`, with absolute links from `~/.agents/skills` and `~/.claude/skills`. Records carry `links` (a list; the old `link` is still read). `--project` skills are unchanged (`.agents/skills` with a relative link from `.claude/skills`). Old global skills are moved on the next `skills`, `doctor` or upgrade, and only into free or already-ours link places.
- **Uninstall:** deletes the home as a whole, except `llmserver.env` and `config/` with `--keep-config`, `models/` unless the models are purged, and `skills/` if a changed skill is kept. Skills are removed first, while the helper Python can still verify them.
- **Tests:** a sixth suite (33 cases: migration with a running 2.0.0 model, moved home, sandbox cannot read the key or the settings, `MODEL_STORE`, foreign folder refused, uninstall with and without `--keep-config`); the other 194 cases pass on the new layout.

### Implementation notes (2.0.0), differences from this design

- **Bubblewrap only, not `systemd-run`:** user scopes cannot apply the filesystem protections without privileges.
- **What the sandbox gives the engine:**
  - a read-only `/`, with `/dev` kept for the GPUs, and a private `/tmp`;
  - an empty home (tmpfs), with a private writable cache per engine at `~/.local/state/llmserver/sandbox/ENGINE`;
  - read-only binds of llmserver's home and the HF cache, and a writable bind of the Ollama store;
  - an engine from PATH: its folder (a venv: the whole venv), but never the home or a parent of it;
  - no HF token, because engines serve offline;
  - no `--new-session`: stopping a model signals its process group, and engines have no controlling terminal anyway;
  - no `--die-with-parent`: that flag killed the server when the starting llmserver exited.
- **No network limit:** without privileges an engine cannot be given its own network and still listen on the host's port. The manual says so.
- **`LLMSERVER_SANDBOX`:** `auto` (default) uses the sandbox when bwrap works and says so when it does not; `on` refuses to start without it; `off` never uses it. `LLMSERVER_SANDBOX_RW` adds writable folders.
- **`eval`:**
  - 9 built-in probes: 5 indirect prompt injections hidden in a document, and 4 system-prompt-secret extractions, each with a random marker word.
  - `--garak` installs garak 0.17.1 on request: hash-locked, with CPU PyTorch (about 1.7 GB), under `LLMSERVER_HOME/tools`. It runs three probes by default (prompt injection, an injection hidden in a document, jailbreaks), or the ones `--probes` names.
  - Results are kept in the check record (the last 20) and shown by `trust show`. They are information only.
- **Tests:** 23 sandbox and eval cases. A real run of llama.cpp in the sandbox could not read `~/.ssh`, and could not write to `/etc` or the model cache. Real built-in probes, plus a garak install and run through llmserver, against that model. Totals: 17 + 72 + 50 + 32 + 23 = 194 cases.

### Implementation notes (1.9.0), differences from this design

- **What is signed:** `SHA256SUMS` (plain text) is signed, not `manifest.json`. `SHA256SUMS` lists every other member, including the manifest. A machine with no llmserver runtime can then check a bundle with `ssh-keygen -Y verify` and `sha256sum -c` alone, before any Python exists. That is what makes `install --from` possible from nothing.
- **Bundle checks:** a bundle must hold only plain files, and exactly the members `SHA256SUMS` lists (no extras, none missing). Links, directories, absolute paths and `..` are refused before anything is unpacked.
- **Offline installs:**
  - **Engine archives** in a bundle still have to match the pinned checksums in the script, so a bundle that is consistent but altered is refused.
  - **Python** comes through uv's own mirror setting (`UV_PYTHON_INSTALL_MIRROR=file://`), so uv still checks its built-in checksums.
  - **Python packages** install with `--no-index --find-links`, against the same hash locks.
- **Wheels are found by hash:** lock files carry hashes but no URLs. Export searches the lock's indexes, PyPI and (for `+cuXXX`/`+rocm` builds) PyTorch's index for files whose SHA-256 is in the lock and whose tags fit the target Python and platform. All 198 packages of the vLLM CUDA lock resolved this way.
- **Same version on both sides:** a bundle with engines must come from the same llmserver version and CPU architecture; for models only, a different version is a warning.
- **Local scanners:**
  - **Used when on PATH:** `clamscan` (all files), `modelscan` (pickles, only with `ALLOW_PICKLE`) and `gitleaks` (model code and skill scripts).
  - **Semgrep is not used:** its rule sets come from a network registry, which does not fit offline use.
  - **A hit is a block** that `--accept-risk` can override; a missing tool is recorded as `not-run`.
- **SBOM:** CycloneDX 1.6. It includes the checked models as `machine-learning-model` components with their file hashes, and validates against the official schema (cyclonedx-python-lib, strict validator).
- **Tests:** 32 bundle cases, including `install --from` with the network blocked (runtime and llama.cpp). Totals: 17 + 72 + 50 + 32 = 171 cases.

### Implementation notes (1.8.0), differences from this design

- **Rules split and added:**
  - Outside domains are a separate rule, S-DOMAINS (note, or ask under strict), not part of S-PATTERN. Most legitimate skills link to outside documentation, so asking every time would train people to say yes.
  - New rules: S-SCRIPTS (note), S-NO-SKILLMD (block) and S-HUB-OFFLINE (block).
- **GitHub check:** llmserver reads the repository's open issues (`/repos/huggingface/skills/issues`, up to 500) and matches the skill name itself, instead of the search API, which is limited to 10 requests a minute. The result is cached for 24 h. `GITHUB_TOKEN` in the settings file is used when set.
- **`--force`:** replaces or removes a skill that was changed locally, or one that llmserver did not install. Without it, both are refused (exit 3).
- **Mirror:** `LLMSERVER_SKILLS_MIRROR` points at a local copy of the marketplace (`marketplace.json` plus `skills/NAME/`), for air-gapped sites and for tests. The copy's administrator is then responsible for what it lists.
- **The `hf-cli` rewrite also drops whole sections:** `hf skills` and `hf extensions` (both refused through llmserver), `hf update`, and the `hf-mount` section (it installs a separate binary). The marketplace's own `hf-cli` entry is hidden in `skills list`, because `skills add hf-cli` always installs the generated one.
- **Installed files:** no `.hf-skill-manifest.json` marker is written, so a separate `hf skills update` leaves llmserver's skills alone. Files are 644 and folders 755.
- **Tests:** 50 skills cases (local mirror and a simulated GitHub), plus a real run against the Hugging Face skills marketplace (`hf-mem`, `huggingface-papers`). The real GitHub issue check could not be reached from the build sandbox.

### Implementation notes (1.7.0), differences from this design

- **LOCK_MODELS defaults to `off`, not `ask`.** Each record already locks the model to its checked commit, and a start always uses the newest record, never upstream `main`. Writing `@commit` into the menu is now only for copying a menu to another machine. For the same reason, rule M-UNLOCKED was not built.
- **More rules:**
  - M-SCAN-WARN: a yellow scan verdict;
  - M-REMOTE-XREPO: auto_map that points at code in another repository;
  - M-NO-WEIGHTS, M-NOT-FOUND, M-ACCESS and M-GATED;
  - M-NO-HASH: no checksum came from the Hub;
  - M-LOCAL-ONLY and M-ATTACHED: Ollama cases.
- **Scanners in the API:** the raw Hub API (`securityFileStatus`) carries all five scanners per file: avScan (ClamAV/Cisco), protectAiScan, jFrogScan, virusTotalScan and pickleImportScan. The gate reads the raw API, because huggingface_hub drops some of them.
- **M-SIZE:**
  - **GGUF:** a ceiling of 16 bits per parameter (32 for F32), not a per-quantization estimate. Measured: small models reach 5-11 bits per weight even at Q2_K and UD-Q8_K_XL.
  - **Safetensors:** the header must account for the file size exactly (M-FORMAT), which catches data appended to a weight file.
- **Format checks after download:** ELF/PE programs, zip files posing as text, invalid safetensors headers and GGUF magic.
- **llama.cpp download flags are refused:** `-hf`, `--model-url` and the like, so the engine cannot fetch unchecked models.
- **Exit codes:** 3 means refused, 4 means an answer or approval is needed.
- **Tests:** 72 trust cases on a fake Hub, the 17 regression cases, and a real run (Hugging Face → `hf` → check → llama.cpp `-m` → chat completion).
Consolidates `notes-model-security.md` (three reviewed documents, the skills plan and the API findings). The recommended answers to decisions 1–9 are applied as defaults; each one is a setting that can be changed.

---

## 1. Purpose and scope

llmserver downloads three kinds of artifact from the internet:

- **engines:** already pinned and hash-checked since 1.6;
- **models:** for vLLM, llama.cpp and Ollama;
- **skills:** for AI agents.

**Trust** is one mechanism that decides, for every model and skill, whether it may be downloaded and whether it may run. It records why in a file that can be audited.

**It protects against:**
- known-malicious files (malware, unsafe pickles);
- code execution through weight formats;
- unreviewed remote code;
- silent upstream changes;
- files tampered with after download;
- lookalike or swapped files;
- for skills: unlisted sources, hidden instructions and obvious overreach.

**It does not protect against** (stated in the manual and in `trust show`):
- backdoored or poisoned weights, which no file scan finds;
- scanner evasion;
- a compromised local user account;
- the content of Ollama-registry models, for which no scan data exists;
- what an agent does with a skill at run time. Enforcing that is the agent runtime's job.

**Non-goals:** a central enterprise service, numeric risk scores, installing system security tools.

## 2. Concepts

| Term | Meaning |
|---|---|
| Artifact | One model (`engine:repo[:quant]@commit`, or an Ollama manifest digest) or one skill (`skill:name`) |
| Inventory | Every file of the artifact at one commit: path, size, SHA-256, class, and Hugging Face's per-file security result |
| Selection | The subset of the inventory the engine actually needs; only these files are downloaded |
| Finding | `{id, severity, file?, message, overridable}`. Severity is `block`, `ask` or `note` |
| Record | A JSON file per artifact: inventory, selection, findings, decision, who, when. Status `accepted` (automatic) or `approved` (explicit) |
| Level | `SECURITY=standard` (default) or `strict` |

**Decision rule:**
- **Any `block`:** refuse. Only overridable blocks can be accepted, one at a time, with `--accept-risk ID`. The acceptance is written into the record.
- **Any `ask`:** ask the user. `--yes` answers. With no terminal and no `--yes`, refuse.
- **`note`:** shown, and recorded.

**There is no score:** a critical finding always wins.

## 3. Settings

| Setting | Values (default **bold**) | Effect |
|---|---|---|
| `SECURITY` | **standard** \| strict | See the level table in section 5 |
| `SECURITY_FLOOR` | **unset** \| standard \| strict | Read **only** from `/etc/llmserver_env`. The user's level cannot go below it, so an administrator can enforce `strict` |
| `ALLOW_PICKLE` | **off** \| on | Allows pickle-only vLLM repositories (their pickle scan must still be clean) |
| `LOCK_MODELS` | **off** \| ask \| on | On first download, rewrite the menu entry to `repo@commit` (built with `off` as the default; see the 1.7.0 notes) |
| `SKILLS` | **ask** \| on \| off | Whether setup offers the `hf-cli` and `llmserver` skills |
| `SCANNERS` | **auto** \| off | Use `clamscan`, `modelscan` and `gitleaks` if they are on PATH (semgrep was dropped: its rules come from a network registry) |
| `TRUST_SIGNERS` | **unset** \| file | ssh `allowed_signers` file; when set, `import` requires a valid signature |

`VERIFY=require` (the existing content-hash check after a download) stays. Under `strict` it cannot be turned off.

## 4. The gate: one pipeline for every model and skill

```
resolve → inventory → select → check → decide → fetch → post-check → record → (start: enforce)
```

1. **Resolve.** Turn `repo[@rev]` into a commit SHA (`model_info(...).sha`) and record the tag it came from. For skills, read the marketplace entry. For Ollama-registry names, read the manifest digest.
2. **Inventory.** Call `list_repo_tree(revision=sha, expand=True)` with the helper runtime: the path, size, LFS SHA-256 and `security` of every file. A file is classed by extension **and** by a magic-byte check after download:
   - weights;
   - config;
   - tokenizer and template;
   - code (`.py`) and scripts (`.sh`);
   - archives;
   - executables (ELF);
   - other.
3. **Select** (per engine, section 6). Only the selected files are downloaded.
4. **Check.** Apply the rules in section 5 to the inventory, but only to the selected files plus any repository-wide facts.
5. **Decide.** Apply the decision rule above. `llmserver trust check` stops here and downloads nothing.
6. **Fetch.**
   - **What is downloaded:** only the selected files, at the exact commit, with the pinned `hf` (`--revision <sha> --include …`).
   - **Hash check:** every file's SHA-256 is checked against the inventory, not only against the cache blob name.
   - **Ollama `hf.co/…` pulls:** the blob digest must equal the LFS SHA-256 of the GGUF that was checked. Otherwise the pull is deleted and refused.
7. **Post-check.**
   - The magic-byte classification.
   - The size check (rule M-SIZE).
   - Optional scanners (`SCANNERS=auto`): a tool that is not installed is recorded as `not-run`, never as `clean`.
8. **Record.** Write `trust/records/<kind>/<id>.json`, mode 644, go-w, written atomically:
   - **Standard:** status `accepted`, written automatically.
   - **Strict:** needs `trust approve`.

   Each record is also logged to `install.log`.
9. **Enforce at start.** Every start, restart and draft model checks its record.
   - **Checked at every start:** the record exists, the commit matches, and the file sizes match.
   - **Strict:** also requires status `approved`, and the model serves offline.
   - **Full SHA-256 check:** `trust verify` and `doctor --deep` rehash every file.

## 5. Rules

| ID | Applies to | Condition | standard | strict | Overridable |
|---|---|---|---|---|---|
| M-SCAN-UNSAFE | Selected file | Hugging Face result `unsafe`, or the antivirus scan not `safe` | block | block | **no** |
| M-PICKLE-IMPORT | Selected pickle | Any import not marked "innocuous" | block | block | **no** |
| M-SCAN-PENDING | Selected file | `queued` or unscanned | ask | block | yes |
| M-PICKLE-ONLY | vLLM | No safetensors weights, only pickle | block unless `ALLOW_PICKLE=on` | block | yes |
| M-REMOTE-CODE | vLLM | `auto_map` in the config, or `--trust-remote-code` in the flags | ask; approval per commit | block until `trust approve --remote-code` | approve only |
| M-SIZE | Weights | Off by more than 15% from the size expected from the parameter count and data type | ask | block | yes |
| M-EXTRA-EXEC | Repository | Archives, ELF files or scripts in the repository (never downloaded) | note | ask | yes |
| M-UNLOCKED | Model | Started from a moving `main` with no locked commit | note | block | no (lock it) |
| M-MOVED | Model | Upstream commit differs from the record | note (`models outdated`) | note; the old commit keeps serving | — |
| M-HUB-OFFLINE | Model | Hub unreachable, so no scan data | note if a record exists, else ask | block if no approved record | yes |
| M-NO-LICENSE | Model | No license in the card | note | ask | yes |
| M-REGISTRY | Ollama registry | No scan data exists | note | ask; approval is per digest | approve only |
| S-NOT-LISTED | Skill | Not in Hugging Face's `huggingface/skills` marketplace | block | block | **no** |
| S-FILETYPE / S-LINK / S-SIZE | Skill file | A binary, a link, a path that leaves the skill folder, a file over 1 MB, or a skill over 5 MB | block | block | **no** |
| S-ISSUE-SECURITY | Skill | Open issue in `huggingface/skills` naming the skill plus security, malicious or vulnerability | block | block | yes |
| S-ISSUE-OPEN | Skill | Other open issues naming it | note | ask | — |
| S-RECENT | Skill | A file uploaded less than 7 days ago | ask | block | yes |
| S-PATTERN | Skill content | Download-and-run, credential paths, outside domains, `eval`/base64 execution | ask | ask | yes |
| S-UNICODE | Skill content | Zero-width or bidirectional characters, or "ignore previous instructions" | ask | block | yes |
| S-TOOLS | Skill | The tools the skill asks the agent to allow (`allowed-tools`) | note | ask | — |
| S-GITHUB-OFFLINE | Skill | The issue check could not run | ask | block | yes |

The manual lists every rule with its reason. `--json` reports every finding.

## 6. Engine behaviour

**vLLM**
- **Selected files:**
  - weights: `*.safetensors`, `*.safetensors.index.json`;
  - config: `config.json`, `generation_config.json`, `preprocessor_config.json`;
  - tokenizer and templates: `tokenizer*`, `special_tokens_map.json`, `*.model`, `*.tiktoken`, `chat_template*`, `*.jinja`.

  `*.py` is included only with an approved remote-code decision. Pickle weights are included only with `ALLOW_PICKLE=on`.
- **Never downloaded:** `.bin`, `.pt`, `.pth`, `.pkl`, `.ckpt`, `.h5`, `.msgpack`, `.ot`, `.tflite`, `.onnx`, archives, executables.
- **Start:** `--revision <sha> --load-format safetensors`, with `HF_HUB_OFFLINE=1` always, because the gate has already downloaded everything. So vLLM can never fetch unvetted files or code at run time.
- **Remote code:** `--trust-remote-code` is added **only** by llmserver from an approved record. A user flag without one is refused, naming rule M-REMOTE-CODE.

**llama.cpp**
- **Selection:** the chosen quant's GGUF (all split parts) plus the matching `mmproj`.
- **Download and start:** with the pinned `hf`, then `llama serve -m <first part> [--mmproj …]` instead of `-hf`. That gives commit locking, per-file checks and offline serving. Draft models go through the same path.

**Ollama**
- **`hf.co/repo:quant`:** the GGUF is gated as for llama.cpp, then `ollama pull`. The pulled blob digest must equal the checked GGUF's SHA-256.
- **Registry names:** the manifest digest is resolved and recorded. Rule M-REGISTRY applies. Ollama's own digest checks stay.

**Existing models (migration):** the first start after the upgrade runs the gate against the files already cached.
- **Hub reachable:** a record is written as usual.
- **Hub offline, standard:** a `legacy` record with a note.
- **Hub offline, strict:** refused until `trust approve`.

## 7. Commands

| Command | Does |
|---|---|
| `llmserver trust check MODEL\|skill:NAME [--json]` | Runs the gate up to "decide"; downloads nothing |
| `llmserver trust approve MODEL [--remote-code] [--accept-risk ID…]` | Runs the gate, fetches, and writes an `approved` record (`approved_by=$USER@host`) |
| `llmserver trust revoke MODEL` | Marks the record revoked; a running model is told to stop |
| `llmserver trust list [--json]` | All records with status, commit and level |
| `llmserver trust show MODEL [--code] [--json]` | Inventory, findings and decision; `--code` prints the remote-code files |
| `llmserver trust verify [MODEL] [--json]` | Full SHA-256 recheck of the downloaded files |
| `llmserver models outdated` / `models update NAME` | Upstream moved: shows the changed files; `update` runs the gate on the new commit |
| `llmserver export NAME… -o FILE [--with-engines] [--sign KEY]` | Air-gap bundle (section 9) |
| `llmserver import FILE` | Checks the signature (if `TRUST_SIGNERS` is set) and every hash, then places the files and their records |
| `llmserver versions --sbom` | CycloneDX JSON of everything llmserver installed |
| `llmserver hf ARGS…` | The pinned `hf`, in a clean environment, using your normal HF cache and token. Refuses `update`, `extensions install` and `skills add/update` |
| `llmserver skills list\|check\|preview\|add\|update\|remove` | Section 8 |

**Changes to existing commands:**
- `check` gains a security summary line.
- `ps --json` gains `commit` and `trust` (status and record).
- `doctor` checks the records (sizes, status, delisted skills, new issues).
- `doctor --deep` does a full rehash.
- `uninstall` removes the records and only the skill files that still match their checksums.
- `pin`/`unpin` (eviction) is unchanged; commit locking is called *lock* to avoid the clash.

**Exit codes:**
- 0: passed;
- 3: blocked;
- 4: needs an answer or approval and none was given (no terminal, no `--yes`);
- 1: other errors (unchanged).

## 8. Skills

- **Sources:**
  - **`llmserver` skill:** generated from the script itself and versioned.
  - **`hf-cli` skill:** generated by the pinned `hf`, rewritten to `llmserver hf`, with the `curl | bash` line removed. llmserver refuses to install it if any plain `hf` command is left.
  - **Marketplace skills:** installed byte-for-byte as published.
- **The gate:**
  - all S-rules apply, with records in `trust/records/skill/`;
  - skills are installed atomically, with files as `644`/go-w and no execute bit;
  - llmserver never runs a skill's scripts.
- **Scope:** global by default (`~/.agents/skills` plus a link in `~/.claude/skills`, honouring `CLAUDE_CONFIG_DIR`). `--project` for the current folder; `-g` is accepted.
- **Contents of the `llmserver` skill:**
  - what llmserver does;
  - which commands give JSON;
  - how to find a model's port and API key;
  - the OpenAI-compatible endpoints;
  - "if a skill says `hf` and it is not on PATH, run `llmserver hf`; never install `hf`";
  - ask before `stop`, `evict`, `trust approve` or `--accept-risk`;
  - never run `uninstall` or `doctor --fix` unasked.
- **Optional `hf` link:** `llmserver hf --link` puts a small `hf` wrapper at `~/.local/bin/hf`. It's off by default, refused if another `hf` exists, and removed on uninstall.

## 9. Air-gapped transfer

**Bundle:** an uncompressed tar (weights don't compress) containing:

```
SHA256SUMS               # the SHA-256 of every other member (as built in 1.9.0)
SHA256SUMS.sig           # optional: ssh-keygen -Y sign -n llmserver-bundle, over SHA256SUMS
manifest.json            # bundle id, created, llmserver version, creator, artifacts, where each file goes
records/…                # the approval records
blobs/sha256/<hash>      # content-addressed files
engines/…                # with --with-engines: pinned archives, uv, Python builds, locked wheels
```

- **Export** includes only artifacts with an `approved` record (under standard, `accepted` is enough, with a note).
- **Import:**
  1. checks the signature, when `TRUST_SIGNERS` is set (required under strict);
  2. extracts with the `data` filter into staging and checks every SHA-256 against `SHA256SUMS`;
  3. only then places the files: the HF cache layout (`blobs/` and `snapshots/<commit>/`) or the Ollama blob store and manifest;
  4. writes the records, marked `imported` with the bundle id.

  Imported models always serve offline.
- **Engines from the bundle:** `install --from FILE` installs uv, the Pythons, the engines and the wheels with no network (`UV_OFFLINE=1 --find-links`), with the same hashes as online.

## 10. Supply-chain extras

- **SBOM (`versions --sbom`):** CycloneDX 1.6 listing uv, Pythons, every locked wheel (name, version, SHA-256, index) and the engine archives (URL, SHA-256). Trivy or Grype can then scan it.
- **Optional scanners (`SCANNERS=auto`):**
  - `clamscan` on downloaded files;
  - `modelscan` on pickles (only when allowed);
  - `semgrep` and `gitleaks` on approved remote code and skill scripts.

  Results go into the record. llmserver installs none of them.
- **Behaviour testing (later, optional):** `llmserver eval NAME` runs garak from its own hash-locked venv, installed on request, plus a small built-in prompt-injection probe. Results are attached to the record as information. llmserver does not claim to detect backdoors.
- **Sandbox:** built in 2.0.0 with bubblewrap only (see the 2.0.0 and 2.7.2 notes); the network stays shared, because the server must listen on the host's port.

## 11. Record format (version 1)

```json
{
  "schema": 1, "kind": "model", "id": "vllm:Qwen/Qwen3-8B@c1899de2…",
  "engine": "vllm", "repo": "Qwen/Qwen3-8B", "ref": "main", "commit": "c1899de2…",
  "status": "approved", "level": "strict",
  "decided_by": "pavan@pnode001", "decided_at": "2026-10-22T14:03:11Z", "llmserver": "1.7.0",
  "remote_code": false, "accepted_risks": [],
  "files": [{"path": "model-00001-of-00004.safetensors", "size": 4967215360, "sha256": "…",
             "class": "weights", "hub_scan": {"status": "safe", "av": "safe", "pickle": "unscanned"},
             "local_scan": {"clamscan": "not-run"}}],
  "skipped": [{"path": "pytorch_model.bin", "class": "weights-pickle", "reason": "not selected"}],
  "findings": [{"id": "M-EXTRA-EXEC", "severity": "note", "file": "convert.sh", "message": "…"}],
  "expected_weight_bytes": 16384000000,
  "source": {"type": "hub", "bundle": null}
}
```

## 12. Robustness

- **Writes:** records are written atomically under the state lock (fd 8), and the gate runs under the start lock (fd 9). Two starts of one model share one gate run.
- **Hub API failures:** retried 3 times, then M-HUB-OFFLINE. Never treated as "clean".
- **GitHub limits:** the GitHub check is cached for 24 h in `trust/cache/` and respects the rate limit (60 requests an hour without a login). An optional `GITHUB_TOKEN` is read from the env file only.
- **Large models:** inventory and checks are metadata-only. Hashing runs once at download (streamed) and again only on `verify` or `--deep`.
- **Parsing:** all JSON is parsed in the helper Python. Nothing from the network is ever evaluated by bash.
- **Interrupted downloads:** staged files are deleted, so nothing is half-recorded.

## 13. Tests (added to the suite)

The fake Hub (`fakehub.py`) gains per-file `security`, commits, `auto_map` and skills buckets. Cases:

- **Scan results:** safe, queued, unsafe, an unsafe pickle import, and pickle-only with `ALLOW_PICKLE` on and off.
- **Selection:** pickle and TensorFlow/Flax/ONNX copies are not downloaded, and `--load-format safetensors` is set.
- **Remote code:** a user `--trust-remote-code` is refused; approval is per commit and is lost on upstream change.
- **Commits:** locking, the `outdated` list, and the menu rewrite.
- **Integrity:**
  - an M-SIZE mismatch;
  - a tampered blob caught by `verify`;
  - a size mismatch refused at start;
  - an Ollama digest that differs from the checked GGUF;
  - a llama.cpp split GGUF started with `-m`;
  - a draft model gated.
- **Levels:** `strict` without approval, with approval, and offline; `SECURITY_FLOOR` cannot be lowered.
- **Bundles:** export/import round trip; a tampered blob, a bad signature and a missing signature under strict are refused.
- **Skills:**
  - every S-rule, the `hf-cli` rewrite check, and atomic install;
  - uninstall removes only unchanged files;
  - `hf` passthrough refusals.
- **Output:** the SBOM validates against the CycloneDX schema; `--json` shapes; exit codes 3 and 4.
- **Regression:** the existing 17 cases still pass, and shellcheck is clean.

## 14. Releases

| Version | Contents |
|---|---|
| **1.7.0** | Gate, inventory and selection, all M-rules, commit locking, remote-code approval, automatic records, `trust check/approve/revoke/list/show/verify`, `models outdated/update`, llama.cpp `-m` switch, Ollama digest match, migration, `SECURITY`/`SECURITY_FLOOR`/`ALLOW_PICKLE`/`LOCK_MODELS` |
| **1.8.0** | Skills (all S-rules), `llmserver hf`, the `llmserver` and `hf-cli` skills, the optional `hf` link, `SKILLS` setting |
| **1.9.0** | `export`/`import`/`install --from`, signatures, `versions --sbom`, optional scanners |
| **2.0.0** | Sandbox, `eval` (garak) |
| **2.1.0** | One folder (`~/.llmserver`), migration, movable home, sandbox hides the home |
| **2.1.1 to 2.7.2** | See the table at the top of this document |

Each release updates `document.md` (ASD-STE100):
- a new chapter "Trust: security of models and skills";
- tasks for check, approve, lock, update, export and import;
- undo steps (revoke, remove a skill, remove records);
- reference tables for the rules, exit codes and record format;
- a limits section listing what is not protected.

## 15. Defaults applied (from decisions 1–9)

1. `SECURITY=standard`; `strict` available, and enforceable with `SECURITY_FLOOR`.
2. Pickle-only vLLM repositories are refused (`ALLOW_PICKLE=off`).
3. llama.cpp switches to the pinned `hf` download plus `-m`.
4. `LOCK_MODELS=ask` (built as `off`: each record already locks the commit).
5. The sandbox comes after 1.9 (built in 2.0.0, hardened in 2.7.2).
6. Approval records are required under `strict`; under `standard` they are written automatically.
7. Air-gapped export and import are included (1.9).
8. Bundle signatures with `ssh-keygen -Y`, required under `strict` when importing.
9. Behaviour testing later, optional.
