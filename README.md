# llmserver

**Run local LLMs on Linux with one command. Every download is checked, every engine is sandboxed, and each model gets its own OpenAI-compatible endpoint.**

![version](https://img.shields.io/badge/version-2.7.3-blue) ![platform](https://img.shields.io/badge/platform-Linux%20x86--64%20%7C%20ARM64-lightgrey) ![license](https://img.shields.io/badge/license-Apache--2.0-green)

llmserver is a single bash script. It installs and runs **llama.cpp**, **vLLM** and **Ollama** for you. Before a model starts, llmserver:

- checks that the model fits your GPUs;
- picks the GPU and port;
- checks every file against Hugging Face's scans and SHA-256 checksums.

Optional add-ons:

- a web chat for your team (**Open WebUI**), with document search;
- private web search (**SearXNG**);
- one gateway address for all your models.

Its design rules:

- **One script.** No Docker, no sudo, no system packages, no system Python.
- **Everything pinned and checked.** Each download is checked against a fixed SHA-256, and Python packages come from hash-locked lists.
- **Safe by default.** The engines run in a sandbox with a clean environment. Models must pass security checks before they start.
- **Self-contained.** Everything lives in `~/.llmserver`. `llmserver uninstall` removes it.

> The full operator manual is [`document.md`](document.md). This README is the short version.

---

## Contents

1. [Quick start](#quick-start)
2. [Web chat for your team (Open WebUI)](#web-chat-for-your-team-open-webui)
3. [One address for all models (gateway)](#one-address-for-all-models-gateway)
4. [Everyday commands](#everyday-commands)
5. [Manage models](#manage-models)
6. [Several models on your GPUs](#several-models-on-your-gpus)
7. [Share models on the network](#share-models-on-the-network)
8. [Documents and web search in the chat](#documents-and-web-search-in-the-chat)
9. [Speed: benchmark and speculative decoding](#speed-benchmark-and-speculative-decoding)
10. [Security](#security)
11. [Computers without internet](#computers-without-internet)
12. [Automation, scripts and AI agents](#automation-scripts-and-ai-agents)
13. [Engines: install, upgrade, roll back](#engines-install-upgrade-roll-back)
14. [Settings](#settings)
15. [Back up, restore and move](#back-up-restore-and-move)
16. [Troubleshooting](#troubleshooting)
17. [Uninstall](#uninstall)
18. [Reference](#reference)
19. [License](#license)

---

## Quick start

### Requirements

| Item | Requirement |
|---|---|
| System | Linux x86-64 or ARM64. WSL2 and containers work |
| Shell and tools | bash 4.4+, curl, tar, gzip, coreutils, findutils, grep, sed, awk |
| C library | glibc 2.35+. glibc 2.39+ for the CUDA llama.cpp builds and for vLLM on AMD |
| GPU (optional) | NVIDIA driver with `nvidia-smi`, or the AMD `amdgpu` driver (vLLM and Ollama on AMD also need `amdkfd`). No GPU: llama.cpp on the CPU |
| Recommended | `bubblewrap` (sandbox), `ss` or `lsof`, `flock`, `setsid`, `timeout` |
| Disk | About 200 MB for the runtime, plus 25–800 MB for llama.cpp, 1.5–2.6 GB for Ollama and 10.5–14 GB for vLLM, plus the size of each model and 10% |

You do not need Python. llmserver installs its own.

### Install

Download `llmserver` and `SHA256SUMS` from the [latest release](../../releases/latest). Then:

```bash
sha256sum -c --ignore-missing SHA256SUMS                  # must show: llmserver: OK
chmod +x llmserver && mkdir -p ~/.local/bin && mv llmserver ~/.local/bin/
llmserver --version                                       # llmserver 2.7.3
```

### Set up

```bash
llmserver setup
```

The setup finds your GPUs and asks a few questions: which engines to use, local or network access, the first port, the context size, and your first models. Press Enter to accept a default. It then shows the install plan and its size, and installs after you type `y`. At the end, it offers to install Open WebUI.

### Run your first model

```bash
llmserver check llama:ORG/REPO-GGUF:Q4_K_M   # will it fit? (no download)
llmserver llama:ORG/REPO-GGUF:Q4_K_M         # check, download, start
llmserver ps                                 # running models, ports, state
```

### Use it

```bash
llmserver chat                               # chat in the terminal
eval "$(llmserver env 1)"                    # sets OPENAI_BASE_URL, OPENAI_API_KEY, LLMSERVER_MODEL
curl -s "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" -H 'Content-Type: application/json' \
  -d '{"model":"'"$LLMSERVER_MODEL"'","messages":[{"role":"user","content":"hello"}]}'
llmserver stop 1                             # stop it
```

Any OpenAI-compatible client works with `http://HOST:PORT/v1`.

### Which engine?

| Engine | Models | Best for |
|---|---|---|
| **llama.cpp** | GGUF repos on Hugging Face, or a GGUF file on your disk | NVIDIA, AMD or CPU. Can split a model between GPU and CPU |
| **vLLM** | Hugging Face repos (safetensors, AWQ, GPTQ, FP8) | Highest throughput. Can split a model across GPUs. Needs the whole model on the GPU |
| **Ollama** | The Ollama library, or `hf.co/...` GGUF repos | Simple model management. Splits between GPU and CPU automatically |

If you leave out the engine, llmserver picks one: a name with `gguf` or `:QUANT` goes to llama.cpp, a name without `/` goes to Ollama, and other Hugging Face names go to vLLM.

---

## Web chat for your team (Open WebUI)

```bash
llmserver webui
```

The first run installs Open WebUI 0.12.3 (about 2.5 GB) and starts it with the gateway. Then:

1. **Create the admin account.** Until the first account exists, Open WebUI listens on `127.0.0.1:3000` only, so nobody on the network can take the admin account first.
   - On the server, open `http://127.0.0.1:3000`.
   - From another computer, run `ssh -L 3000:127.0.0.1:3000 USER@HOST` first, then open the same address.
2. **Open it to the network:** `llmserver webui restart`. It now listens on `0.0.0.0:3000`. If other computers still cannot connect, open the port in the firewall.
3. **Add people** in *Admin Panel → Users*, or turn on sign-ups (new accounts wait for your approval).
4. **Start models** with `llmserver MODEL`. Open WebUI shows every running model at once.

| Task | Command |
|---|---|
| Status / stop | `llmserver webui status` · `llmserver webui stop` |
| Apply changed `webui_*` settings | `llmserver webui restart` |
| This computer only | `llmserver config set webui_host 127.0.0.1` |
| Another port | `llmserver config set webui_port 8443` |
| HTTPS with your certificate | `llmserver config set webui_tls_cert /path/cert.pem` and `webui_tls_key /path/key.pem` (mode 600), then restart |
| Start at login | `llmserver autostart on webui` |

How llmserver sets up Open WebUI:

- Everyone needs an account.
- It connects only through the gateway.
- It is offline at run time. Its embedding and reranker models are pinned and checked.
- Community sharing and update checks are off.
- It runs in the sandbox and can write only to `~/.llmserver/webui`.
- It gets none of your tokens.
- It cannot fetch pages on your private network.

> Open WebUI has its own license. Keep its branding if more than 50 people use it in 30 days, unless you have an enterprise license.

---

## One address for all models (gateway)

Each model has its own port. The gateway puts all of them behind `http://127.0.0.1:8000/v1`. Clients choose a model by name, as `GET /v1/models` lists it.

```bash
llmserver gateway start          # Open WebUI starts it for you
```

**Start models on demand.** The gateway can start a menu model when a request names it, and stop it again when it is idle:

```bash
llmserver config set gateway_autoload on
llmserver config set gateway_idle_stop 30      # minutes; 0 = never
llmserver gateway stop && llmserver gateway start
```

The gateway starts only models that already passed the security checks on this computer. It never downloads a model or answers a security question, so start a new model once yourself.

| Topic | Details |
|---|---|
| Network | `llmserver config set gateway_host 0.0.0.0`. Clients on the network must send the API key |
| Limits | 64 requests at once (more get 503). A client that sends nothing for 60 s is disconnected |
| Monitoring | `curl -s http://127.0.0.1:8000/metrics` (Prometheus): requests, in flight, models running, models started by the gateway |
| Log | `~/.llmserver/logs/gateway.log`: one line per request, never the message text |

---

## Everyday commands

| Command | What it does |
|---|---|
| `llmserver` | Shows the menu and starts the model you pick |
| `llmserver MODEL` | Starts a model (menu number or name) |
| `llmserver ps` | Running models |
| `llmserver stop MODEL` · `stop all` | Stops models |
| `llmserver chat [MODEL] ["question"]` | Chats in the terminal. With a question or a pipe, it prints only the answer |
| `llmserver env [MODEL]` | Prints the client variables |
| `llmserver logs MODEL` · `llmserver log` | Model output · llmserver activity (`--errors` for problems only) |
| `llmserver check [MODEL]` | Hardware and a fit verdict, without a download |
| `llmserver capacity` | GPU memory, what is on each GPU, what still fits |
| `llmserver doctor [--fix]` | Health check and repair |

Useful start options:

| Option | Effect |
|---|---|
| `--ctx N` · `--ctx fit` | Context size, or the largest context that fits in free GPU memory |
| `--port N` | A fixed port (a model keeps its port anyway) |
| `--restart` | Restarts a running model with new options |
| `--foreground` | Shows the server output. Ctrl-C stops it |
| `-n` | Shows the plan and changes nothing |
| `-- FLAGS` | Passes flags to the engine |

Chat commands: `/think` (show reasoning), `/reasoning low|medium|high|none`, `/system TEXT`, `/clear`, `/retry`, `/save FILE`, `"""` for multi-line messages, `/exit`.

---

## Manage models

### The menu

```bash
llmserver model add vllm:Qwen/Qwen3-8B                        # checks that the name exists
llmserver model add llama:ORG/REPO-GGUF:Q4_K_M -- -c 65536    # with engine flags
llmserver model list
llmserver model rm 4 [--purge]                                # --purge also deletes its files
llmserver model edit                                          # edit in $EDITOR, then validated
```

The menu is `~/.llmserver/config/models`, with one model per line. After the model name, a line can hold:

- engine flags;
- `port=N`;
- `draft=MODEL` and `draft-n=N`.

Quote values that have spaces. A model you start by name is added to the menu automatically.

### Model names

| Name | Engine |
|---|---|
| `llama:ORG/REPO[:QUANT][@COMMIT]` | llama.cpp (Q4_K_M by default) |
| `vllm:ORG/REPO[@REVISION]` | vLLM |
| `ollama:NAME[:TAG]` · `ollama:hf.co/ORG/REPO[:QUANT]` | Ollama |
| `~/models/file.gguf` | llama.cpp, from your disk |

### A GGUF file on your disk

```bash
llmserver model add ~/models/MyModel-Q8_0.gguf
llmserver 3
```

On the first start, llmserver:

- checks that the file is a real GGUF file;
- records its SHA-256;
- runs the local scanners, if they are installed.

It never copies, moves or deletes your file. Split files (`-00001-of-0000N`) and a vision projector (`mmproj-*.gguf`) in the same folder are found automatically.

### Download, update and clean up

```bash
llmserver download MODEL        # check and download now; running models keep running
llmserver models outdated       # which models changed upstream
llmserver models update MODEL   # check and download the new commit
llmserver MODEL --restart       # use it
llmserver models prune -n       # show the files of older commits
llmserver models prune          # delete them (files still in use are kept)
```

A model never moves to a new commit by itself. To pin a menu entry to a commit, add `@COMMIT` to it, or run `llmserver config set lock_models on`.

### Where model files live

By default, model files go to the standard caches (`~/.cache/huggingface/hub` and `~/.ollama/models`), so other tools can use them too.

To keep them in `~/.llmserver/models` and still let other tools find them:

```bash
llmserver config set model_store private
llmserver models share on -n     # show the plan
llmserver models share on        # adds HF_HUB_CACHE and OLLAMA_MODELS to ~/.profile (and bash/zsh files)
. ~/.profile                     # or log in again
llmserver models share           # status
```

`models share off` removes these lines, and `uninstall` removes them too. Each line applies only while its folder exists. Models in the old caches are not moved; each one downloads again on its next start. systemd services and the fish shell don't read these files: give a service `Environment=` in `systemctl --user edit`, and in fish use `set -Ux`.

---

## Several models on your GPUs

llmserver plans GPU memory for you. It places each new model on the GPU with room for it, next to the models that are already running.

```bash
llmserver 1 && llmserver 2    # each model gets the next free port and the best GPU
llmserver capacity            # per-GPU memory and what fits ("fits now", "fits after stopping", "never fits")
```

**When the GPUs are full,** llmserver follows `evict`:

| `llmserver config set evict …` | In a terminal | Without a terminal |
|---|---|---|
| `ask` (default) | Shows which models must stop, and asks | Refuses |
| `lru` | Stops the least recently used models | Same |
| `never` | Refuses | Same |

For a single start, you can override this:

- `--evict`: stop models without asking.
- `--offload`: put part of the model on the CPU instead of stopping anything.

If a new model fails to start, llmserver restarts the models it stopped, on the same ports.

| Task | Command |
|---|---|
| Protect a model from eviction | `llmserver pin 1` · `llmserver unpin 1` |
| One model at a time | `llmserver config set max_models 1` and `evict lru` |
| Use only some GPUs | `llmserver config set gpus 0,2` (masks such as `CUDA_VISIBLE_DEVICES` are also obeyed) |

**Verdicts:**

| Verdict | Meaning |
|---|---|
| `GOOD` | Fits on the GPU with 10% to spare |
| `TIGHT` | Fits, but long prompts may fail |
| `OFFLOAD` | Part runs on the CPU (slow) |
| `CPU` | No usable GPU (slow) |
| `NO` | Will not run on this computer |
| `UNKNOWN` | Size unknown |

`vram_guard` (`off`, `warn`, `enforce`, `strict`) decides which verdicts are refused.

---

## Share models on the network

```bash
llmserver config set host 0.0.0.0
llmserver MODEL --restart
llmserver env MODEL           # clients need OPENAI_API_KEY
```

Then open the ports in your firewall.

| Topic | Details |
|---|---|
| API key | Required off this computer. `llmserver key show` shows where it is. `llmserver key new` replaces it; restart the models afterwards |
| Shared computer | `llmserver config set auth key`: every client needs the key, also on `127.0.0.1`. `doctor` warns when other accounts can log in |
| Ollama | Cannot check a key, so llmserver keeps it off the network (`allow_insecure 1` overrides this; use it only on a trusted network) |
| Encryption | The key does not encrypt traffic. On untrusted networks, put a TLS proxy in front |

---

## Documents and web search in the chat

### Hybrid document search (on by default)

Open WebUI finds passages in your documents in three steps:

1. A keyword search (BM25) and a meaning search each find 20 passages.
2. The two lists are merged.
3. A reranker scores each passage, and the best 6 go to the model.

| `llmserver config set webui_reranker …` | Reranker | Use it for |
|---|---|---|
| `fast` (default) | ms-marco-MiniLM-L6-v2 on the CPU (91 MB) | English. About 1.5 s for 20 passages |
| `best` | bge-reranker-v2-m3 on llama.cpp (0.6 GB), pinned and checked | Many languages and the best order. Use a GPU |
| `off` | None | Very little memory |

Apply the change with `llmserver webui restart`. With `best`, the reranker shows up in `llmserver ps` as `rerank`. Set `webui_rag vector` to use only the meaning search.

### Web search (off by default)

```bash
llmserver config set webui_search on
llmserver webui restart
```

The first run installs a private SearXNG (about 150 MB), pinned to a checked commit. It runs on `127.0.0.1:8888`, in the sandbox, and only Open WebUI can use it. In a chat, click **Web Search**.

> Search questions go to public search engines, and Open WebUI reads the result pages. Turn this on only if your users are allowed to do that.

To choose the search engines, copy `~/.llmserver/webui/searxng/settings.yml` to `~/.llmserver/config/searxng.yml` and edit the copy.

---

## Speed: benchmark and speculative decoding

```bash
llmserver bench 1                 # TTFT, prompt and decode speed, TPOT, ITL p50/p90/p99, E2E, plus one real question
llmserver bench 1 --reasoning low # measure the real question at a reasoning level
llmserver bench --history         # compare runs
```

After one benchmark, `check` and `list` estimate the speed of your other models.

**Speculative decoding** uses a small draft model from the same family:

```bash
llmserver llama:ORG/BIG-GGUF:Q4_K_M --draft llama:ORG/SMALL-GGUF:Q4_K_M
llmserver vllm:ORG/REPO --draft ngram        # vLLM prompt lookup, no draft model
```

Compare with `--no-draft`. Ollama has no speculative decoding.

---

## Security

### What happens before every download

1. llmserver resolves the exact commit on Hugging Face.
2. It reads each file's size, SHA-256 and scan results (ClamAV, pickle scan, Protect AI, JFrog, VirusTotal).
3. It selects only the files the engine needs. For vLLM that means safetensors and config files: no pickle files, code or archives.
4. It applies the [rules](#security-rules) and decides: **pass**, **ask** or **refuse**.
5. It downloads at that commit and checks size, format and SHA-256 again.
6. It writes a **check record**. Later starts use the record, with no network needed.

At each start, llmserver compares each file's size and modification time with the record. If only the time changed, it hashes the file again.

### Security levels

| | `standard` (default) | `strict` |
|---|---|---|
| File flagged unsafe | Refused | Refused |
| File not scanned yet | Asks | Refused |
| Model runs its own Python code | Asks for approval | Refused without approval |
| Start needs | A check record | An **approved** check record |

```bash
llmserver config set security strict
```

An administrator can enforce a minimum level for all users with `LLMSERVER_SECURITY_FLOOR=strict` in a root-owned `/etc/llmserver_env`.

### Commands

| Command | What it does |
|---|---|
| `llmserver trust check MODEL` | Runs the checks without downloading |
| `llmserver trust approve MODEL [--remote-code]` | Approves a model, and optionally its Python code (for one commit) |
| `llmserver trust show MODEL [--code]` | Shows the record: files, hashes, scans, findings. `--code` shows the Python code |
| `llmserver trust verify` | Hashes all downloaded files again (exit 3 if one changed) |
| `llmserver trust revoke MODEL` | Withdraws the approval and stops the model |
| `--accept-risk ID` | Accepts one refusable finding. It stays in the record |

Without a terminal, a warning ends with exit status 4. Add `--yes` to accept warnings. It never approves model code.

### Sandbox

Install bubblewrap (`sudo apt install bubblewrap`) and run `llmserver doctor` to confirm it works. Each engine, Open WebUI and SearXNG then run in a sandbox with:

- a read-only system;
- an empty home: no SSH keys, cloud credentials or your files;
- read-only model files;
- no view of your settings, API key or logs;
- their own process namespace;
- no session sockets (D-Bus, SSH agent, keyring);
- a private `/tmp`.

With or without the sandbox, engines get a **clean environment**: only `PATH`, locale and GPU/engine variables, plus what is in `llmserver.env`. Cloud keys and tokens in your shell never reach an engine.

`LLMSERVER_SANDBOX` takes `auto` (default), `on` (refuse to start without the sandbox) or `off`.

### More tools

| Tool | Command |
|---|---|
| Probe a model for prompt injection and system-prompt leaks | `llmserver eval MODEL` (9 built-in probes) · `--garak` for more |
| Software bill of materials (CycloneDX 1.6) | `llmserver versions --sbom > sbom.json` |
| Local scanners | `clamscan`, `modelscan` and `gitleaks` run automatically if they are on `PATH`. A missing scanner is recorded as `not-run` |
| Check the script | `llmserver doctor` prints the script's SHA-256 |

### Limits

The checks cannot find:

- a model trained to misbehave (a backdoor);
- a brand-new attack;
- harmful plain-language text in a skill.

The Ollama library publishes no scan results. The sandbox does not block the network. An engine can read its own API key. Anyone with your user rights can change files and records.

---

## Computers without internet

**Move checked models** from computer A (online) to computer B (offline). Both must run the same llmserver version.

```bash
# On A
llmserver export MODEL1 MODEL2 -o models.tar --sign ~/.ssh/id_ed25519
# On B
llmserver import models.tar
```

**Install the engines offline.** Computer A must have the same CPU architecture and GPU type as B.

```bash
# On A
llmserver export -o engines.tar --with-engines
# On B
llmserver install --from engines.tar
```

A bundle is refused if any file is changed, missing or extra. To require signatures, set `LLMSERVER_TRUST_SIGNERS` to an ssh `allowed_signers` file (with namespace `llmserver-bundle`). With `strict`, signatures are required.

**Behind a firewall,** use a mirror. Every file is still checked against its checksum.

| Setting | For |
|---|---|
| `LLMSERVER_GITHUB` | GitHub release files |
| `UV_INDEX_URL` | Python packages |
| `UV_PYTHON_INSTALL_MIRROR` | Python builds |
| `HTTPS_PROXY`, `SSL_CERT_FILE` | A proxy and its certificate |

**Engine file you downloaded yourself:** `llmserver install llama --from ~/Downloads/llama-…tar.gz`. The file is accepted only if its SHA-256 matches the pinned release.

---

## Automation, scripts and AI agents

| Task | How |
|---|---|
| JSON output | Add `--json` to `ps`, `check`, `capacity`, `list`, `doctor`, `versions`, `trust`, `models`, `skills`, `eval`, `config` or `bench`. JSON goes to stdout, messages to stderr |
| Health check | `llmserver doctor` exits with 1 while a problem remains |
| Start models at login | `llmserver autostart on 1 2` (systemd user services). Add `loginctl enable-linger $USER` to start at boot |
| One-shot answers | `llmserver chat 2 "question"` or `cat notes.txt \| llmserver chat 2` |
| Hugging Face CLI | `llmserver hf auth login`, or any other `hf` command. `llmserver hf --link` adds `~/.local/bin/hf` |

**Exit status:**

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Error |
| `3` | Refused by security checks, a changed file, or a bad bundle |
| `4` | Needs an answer or an approval |

### Skills for AI agents

Skills teach agents such as Claude Code how to use llmserver. llmserver keeps them in `~/.llmserver/skills`, and links them from `~/.agents/skills` and `~/.claude/skills`.

```bash
llmserver skills add                 # the generated llmserver and hf-cli skills
llmserver skills check NAME          # check a Hugging Face marketplace skill
llmserver skills preview NAME        # read it
llmserver skills add NAME [--project]
llmserver skills update | remove NAME
```

Marketplace skills are checked before install:

- only text files, with size limits;
- no security issues reported on GitHub;
- no download-and-run commands, credential reads, hidden characters or injected instructions;
- every file is hashed.

llmserver never runs skill scripts. Read a skill before you install it.

---

## Engines: install, upgrade, roll back

```bash
llmserver install all -n          # show the plan
llmserver install llama ollama    # or: vllm, runtime, webui, searxng
llmserver versions                # what is installed
llmserver upgrade                 # pinned versions of everything installed
llmserver upgrade llama --latest  # newest release, only if it publishes SHA-256 sums
llmserver rollback llama          # previous build (run again to go forward)
```

Running models keep their engine until you restart them.

| Engine | Pinned version | Build |
|---|---|---|
| llama.cpp | b11798 | CUDA 13.4 (driver 580+, compute capability 7.5+), CUDA 12.8 (driver 570+), Vulkan (AMD, mixed GPUs, older drivers), CPU |
| Ollama | v0.40.7 | Linux, plus ROCm libraries for AMD |
| vLLM | 0.32.0 | CUDA (NVIDIA), ROCm 7.2.3 (AMD x86-64). Needs a GPU |
| Runtime | uv 0.13.4, Python 3.14.8 | One build for all |

**Use your own engine:** `llmserver config set llama_source /opt/llama.cpp/build/bin`. The value can be `system` (the copy in `PATH`), a folder or a program. `vllm_source` and `ollama_source` work the same way. llmserver never changes your engine.

**In a container:** pass the GPUs (NVIDIA container toolkit, or `--device /dev/kfd --device /dev/dri`) and mount `/sys` read-only. llmserver follows the container's memory limit and GPU masks.

---

## Settings

```bash
llmserver config                  # every setting and where its value comes from
llmserver config set port 9000    # validated before it is saved; names are case-insensitive, without LLMSERVER_
llmserver config set HF_TOKEN hf_...   # engine variables in capitals reach every engine
llmserver config unset port
llmserver config edit
```

Values are read in this order; later ones win: `/etc/llmserver_env` → `~/.llmserver/llmserver.env` (mode 600) → environment → command options. llmserver uses `/etc/llmserver_env` only if it belongs to root (or you) and nobody else can change it.

<details>
<summary><b>Most used settings</b></summary>

| Setting | Default | Meaning |
|---|---|---|
| `host` · `port` · `port_span` | `127.0.0.1` · `8080` · `100` | Model address and port range |
| `auth` | `auto` | `auto` (key off this computer), `key` (always) or `none` |
| `ctx` | `32768` | Context: a number, `fit` or `auto` |
| `max_models` · `evict` | `0` · `ask` | Model limit (`0` = no limit), and what to do when the GPUs are full |
| `gpus` | all | GPUs llmserver may use |
| `engines` · `default_engine` | installed | Enabled engines, and the engine for names that more than one engine can run |
| `llama_flags` · `vllm_flags` | `--jinja` · empty | Flags for every start |
| `vram_guard` · `vram_reserve_mb` | `enforce` · `512` | Fit enforcement, and memory kept free on each GPU |
| `security` · `sandbox` · `scanners` | `standard` · `auto` · `auto` | Security level, bubblewrap, local scanners |
| `lock_models` | `off` | Write the checked commit into the menu |
| `model_store` | `shared` | `shared` caches or `private` (`~/.llmserver/models`) |
| `gateway_host` · `gateway_port` | `127.0.0.1` · `8000` | Gateway address |
| `gateway_autoload` · `gateway_idle_stop` | `off` · `0` | Start models on demand, and the idle stop in minutes |
| `webui_host` · `webui_port` · `webui_auth` | `0.0.0.0` · `3000` · `on` | Open WebUI |
| `webui_tls_cert` · `webui_tls_key` | empty | HTTPS for Open WebUI |
| `webui_rag` · `webui_reranker` | `hybrid` · `fast` | Document search |
| `webui_search` · `webui_search_port` | `off` · `8888` | SearXNG web search |
| `llama_source` · `vllm_source` · `ollama_source` | `managed` | Use your own engine |
| `llama_build` | `auto` | `cpu`, `vulkan`, `cuda-12.8`, `cuda-13.4` |
| `ready_timeout` · `stop_timeout` | `900` · `20` | Seconds |
| `auto_install` | `on` | Install the runtime when needed |

The full list is in [`document.md` §6.4](document.md#64-settings). `LLMSERVER_HOME` can be set only in the environment or in `/etc/llmserver_env`.

</details>

---

## Back up, restore and move

```bash
llmserver backup -o ~/llmserver-backup.tar.gz     # mode 600; holds your keys, so keep it safe
llmserver stop all && llmserver webui stop && llmserver gateway stop
llmserver restore ~/llmserver-backup.tar.gz
```

A backup holds the settings, menu, keys, check records, ports, pins and skills, plus Open WebUI's accounts, chats and uploads. It does not hold engines or models; these download again.

**Move the home** to another disk:

```bash
llmserver stop all
mv ~/.llmserver /data/llmserver
export LLMSERVER_HOME=/data/llmserver    # in your shell profile, or in /etc/llmserver_env
llmserver list                           # adjusts paths and links
llmserver models share on                # only if you share the model store
```

---

## Troubleshooting

Start with `llmserver doctor`, then `llmserver doctor --fix`. After a crash or power loss, `doctor --fix` takes back running servers, finishes interrupted starts and restarts models that a start had stopped.

| Symptom | Fix |
|---|---|
| Open WebUI only on `127.0.0.1` | First start: create the admin account, then run `llmserver webui restart`. Also check `llmserver config get webui_host` |
| Network clients can't connect | Open the port in the firewall. Check `host` (models), `webui_host` and `gateway_host` |
| Clients get 401 | Send the key: `llmserver env MODEL` |
| `security warning … no terminal` (exit 4) | Run it in a terminal, or add `--yes` |
| `failed the security check` (exit 3) | Read the `[ID]`. Use another model, or `--accept-risk ID` if it is offered |
| `runs its own Python code` | Read it with `trust show MODEL --code`, then `trust approve MODEL --remote-code` |
| `M-ACCESS` (gated model) | Accept the terms on Hugging Face, then `llmserver hf auth login` |
| `needs these models stopped … no terminal` | Use `--evict` or `--offload`, or stop a model |
| `will not run well enough` | `llmserver check MODEL`, then use a recommended quant or context |
| vLLM: too little memory for the KV cache | Use a smaller `--ctx`, or add `-- --gpu-memory-utilization 0.9` |
| `has no Q4_K_M GGUF` | Add a quant from the list shown, for example `:Q8_0` |
| `N%-on-cpu` in `ps` (Ollama) | Stop other models, then `llmserver MODEL --restart` |
| `needs glibc 2.39` | `llmserver config set llama_build vulkan`, or use your own engine |
| `bubblewrap does not work here` | Allow user namespaces, or set `sandbox auto` |
| Documents fail in Open WebUI | The best reranker stopped: `llmserver webui restart` |
| Other tools don't see the models | `llmserver models share on`, then log in again |
| `writable by your group or other users` | `llmserver doctor --fix` |
| A model stays `starting` | `llmserver logs MODEL`; raise `ready_timeout` for slow loads |

All messages are listed in [`document.md` §5.3](document.md#53-problems-and-corrective-actions).

---

## Uninstall

```bash
llmserver uninstall -n     # show the plan
llmserver uninstall        # asks whether to delete the models it downloaded
```

The uninstall removes:

- the systemd units;
- the models it started (they are stopped);
- the skills and links it installed, if you didn't change them;
- the `models share` lines;
- `~/.llmserver`;
- the script.

| Option | Effect |
|---|---|
| `--purge-models` | Also deletes the models llmserver downloaded (never other tools' files) |
| `--keep-config` | Keeps the settings, the menu, the API key and Open WebUI's data |
| `--keep-script` | Keeps the script |
| `--yes` | No questions |

The uninstall never touches:

- engines you installed;
- your own GGUF files;
- other tools' models;
- `/etc/llmserver_env`;
- skills you changed.

---

## Reference

<details>
<summary><b>All commands</b></summary>

| Command | Function |
|---|---|
| `[MODEL]` | Start a model, or show the menu |
| `ps` · `capacity` · `check [MODEL]` · `list` | State, GPU memory, verdicts, menu |
| `model list\|add\|rm\|purge\|edit` | Manage the menu |
| `stop MODEL\|all` · `pin` · `unpin` | Stop and protect |
| `chat [MODEL] [MESSAGE]` · `env [MODEL]` | Terminal chat · client variables |
| `logs [MODEL]` · `log` | Model output · activity log |
| `bench [MODEL]` | Measure speed |
| `download [MODEL]` | Check and download |
| `webui [start\|stop\|restart\|status]` | Open WebUI |
| `gateway [start\|stop\|status]` | One address for all models |
| `config [list\|get\|set\|unset\|edit]` · `key show\|new` | Settings · API key |
| `autostart [list\|on\|off] MODEL…` | Start at login |
| `trust check\|approve\|revoke\|show\|verify\|list` | Security records |
| `models outdated\|update\|prune\|share` | Upstream changes, cleanup, store sharing |
| `backup [-o FILE]` · `restore FILE` | Back up and restore |
| `skills list\|check\|preview\|add\|update\|remove` | AI-agent skills |
| `export` · `import` · `install --from` | Offline bundles |
| `eval [MODEL]` · `versions [--sbom]` | Probes · versions and SBOM |
| `hf ARGS…` | The pinned Hugging Face CLI |
| `setup` · `install` · `upgrade` · `rollback` · `doctor` · `uninstall` | Life cycle |

</details>

<details>
<summary><b>Files in <code>~/.llmserver</code> (mode 700)</b></summary>

| Path | Contents |
|---|---|
| `llmserver.env` | Settings (mode 600) |
| `config/models` · `config/api-key` · `config/tuning` | Menu · API key (mode 600) · tuning rules |
| `logs/` | `install.log`, `models/`, `gateway.log`, `webui.log`, `searxng.log`, `garak.log` |
| `state/` | Running records, ports, pins, rollback records, sandbox caches, chat history |
| `trust/records/` | Check records of models and skills |
| `engines/` · `runtime/` · `python/` · `uv/` · `tools/` | Installed software |
| `webui/` | Open WebUI data (accounts, chats, uploads) |
| `skills/` | Agent skills |
| `models/` | Model files, with `model_store private` only |

Outside the home, llmserver writes only when you ask:

| What | Command |
|---|---|
| `~/.agents/skills` and `~/.claude/skills` links | `skills add` |
| `~/.config/systemd/user/` units | `autostart on` |
| `~/.local/bin/hf` | `hf --link` |
| A marked block in `~/.profile` (and bash/zsh files) | `models share on` |

</details>

<a name="security-rules"></a>
<details>
<summary><b>Security rules</b></summary>

| ID | Condition | standard | strict | How to continue |
|---|---|---|---|---|
| M-SCAN-UNSAFE | A file is flagged unsafe | Refuse | Refuse | Not possible |
| M-PICKLE-IMPORT | A pickle imports a dangerous object | Refuse | Refuse | Not possible |
| M-SCAN-WARN · M-SCAN-PENDING | A scan warning · not scanned yet | Ask | Refuse | `--accept-risk` |
| M-PICKLE-ONLY | vLLM model has only pickle weights | Refuse | Refuse | `allow_pickle on` (standard) |
| M-REMOTE-CODE | The model runs its own Python code | Approval | Refuse | `trust approve --remote-code` |
| M-REMOTE-XREPO | Code from another repository | Refuse | Refuse | Not possible |
| M-SIZE | Weights larger than the parameter count explains | Ask | Refuse | `--accept-risk` |
| M-EXTRA-EXEC · M-NO-LICENSE | Code in the repo (not downloaded) · no license | Note | Ask | `--accept-risk` |
| M-REGISTRY | Ollama library model (no scans) | Note | Ask | `trust approve` |
| M-HUB-OFFLINE | Hugging Face unreachable | Ask | Refuse | `--accept-risk` |
| M-ACCESS · M-NOT-FOUND · M-NO-WEIGHTS | Gated · not found · nothing vLLM can load | Refuse | Refuse | Log in · fix the name |
| M-HASH · M-FORMAT · M-DIGEST | Downloaded file differs from the checked file | Refuse | Refuse | Not possible |
| M-LOCAL-FILE | GGUF file on your disk | Ask | Approval | `--accept-risk` |
| M-LOCAL-SCAN · S-LOCAL-SCAN | A local scanner found a problem | Refuse | Refuse | `--accept-risk` |
| S-* | Skill rules (listing, file types, GitHub issues, patterns, hidden text) | see manual | | |

Every rule is described in [`document.md` §6.11](document.md#611-security-rules).

</details>

---

## License

llmserver is licensed under **Apache-2.0**.

llmserver does not include the programs it installs. It downloads them on your computer, and each keeps its own license:

- **Open WebUI:** BSD-3-Clause with a branding clause. Keep the "Open WebUI" branding if more than 50 people use it in 30 days, unless you have an enterprise license.
- **SearXNG:** AGPL-3.0. It is installed from its pinned source with its license; llmserver adds only a version file.
- **llama.cpp** (MIT), **vLLM** (Apache-2.0), **Ollama** (MIT), **uv** (Apache-2.0 or MIT).
- **Each model** has its own license. `llmserver trust show MODEL` shows it.
