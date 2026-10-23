# llmserver

**One script for Linux that operates local large language models (LLMs) with llama.cpp, vLLM or Ollama. Each model gets its own OpenAI-compatible API. A web chat with document search and private web search is included.**

Version 2.7.1. This document uses ASD-STE100 Simplified Technical English. Instructions are short and in the active voice, and there is one action in each step.

llmserver is one bash script. It installs all that it needs into one folder, `~/.llmserver`. It does not use Docker or sudo, and it does not install system packages. Each download has a fixed version and a fixed SHA-256 checksum.

---

## Features

| Feature | What llmserver does | Section |
|---|---|---|
| Three engines | Operates GGUF models on llama.cpp, Hugging Face models on vLLM, and Ollama models. Each model gets its own port and `/v1` endpoint | [1.1](#11-what-llmserver-is), [3.4](#34-start-a-model) |
| Memory plan | Calculates the memory that a model needs at your context size. Tells you if the model will operate well, operate slowly or not operate | [3.1](#31-examine-the-computer-and-the-models), [6.6](#66-verdicts) |
| GPU placement | Finds the best GPU next to the models that operate already. Shows which models must stop when the GPUs are full. Starts them again if a start fails | [3.5](#35-start-more-than-one-model), [3.8](#38-decide-what-to-do-when-the-gpus-are-full) |
| Model security checks | Reads the Hugging Face scan results before the download. Downloads only the files that the engine needs, at a fixed commit. Does a check of the format and the SHA-256 of each file. Refuses pickle files and remote code unless you permit them | [4](#4-control-the-security-of-models-and-skills) |
| Two security levels | `standard` asks about warnings. `strict` needs your approval for each model | [4.2](#42-select-the-security-level) |
| Sandbox | Operates the engines, Open WebUI and SearXNG in bubblewrap: a read-only system, an empty home folder, no access to your API key or settings | [4.17](#417-isolate-the-engines) |
| Self-contained installation | Installs uv, a private Python and the engines into `~/.llmserver`. Does not change your operating system | [1.3](#13-self-contained-installation) |
| Your own engines and files | Uses an engine that you installed, a release file that you downloaded, or a GGUF file on your computer | [3.18](#318-use-an-engine-that-you-installed), [3.19](#319-install-an-engine-from-a-file-that-you-downloaded), [3.29](#329-use-a-gguf-file-on-your-computer) |
| Terminal chat | `llmserver chat` streams the answers, shows the speed, hides or shows the thinking, and sets the reasoning level | [3.28](#328-chat-with-a-model) |
| Speed measurement | `llmserver bench` measures TTFT, prefill, decode, TPOT, ITL percentiles and E2E latency, with thinking and answer tokens separate | [3.13](#313-measure-the-speed) |
| Speculative decoding | Uses a draft model (llama.cpp, vLLM) or prompt lookup (vLLM) | [3.14](#314-use-speculative-decoding) |
| Gateway | One OpenAI-compatible address for all running models. Selects the model by its name | [3.33](#333-use-one-address-for-all-models-the-gateway) |
| Web chat (Open WebUI) | Installs and operates Open WebUI for the people on your network: accounts, an administrator, approval of new accounts, all running models | [3.30](#330-give-people-on-the-network-a-web-chat-open-webui) |
| Hybrid document search | Keyword search (BM25) and meaning search (vectors), then a reranker. A fast reranker on the CPU, or bge-reranker-v2-m3 on llama.cpp for many languages | [3.31](#331-find-answers-in-documents-hybrid-search) |
| Private web search | A private SearXNG on `127.0.0.1` gives web search to the chat, without cookies | [3.32](#332-search-the-web-from-the-chat-searxng) |
| Network access | Puts models on the network with an API key. Makes new keys | [3.16](#316-make-the-models-available-on-the-network) |
| Automatic start | Starts models and Open WebUI when you log in (systemd user services) | [3.22](#322-start-models-automatically-after-login) |
| Settings without an editor | `config`, `model`, `key` and `autostart` commands do a check of each change before they save it | [3.3](#33-add-remove-and-change-the-models-in-the-menu), [3.12](#312-change-the-settings) |
| Updates and rollback | Installs the specified versions or the newest verified versions. Goes back to the previous build | [3.20](#320-update-the-engines), [7.4](#74-go-back-to-the-previous-engine-build) |
| Computers without a network | Exports models and engines into a signed bundle. Imports the bundle on a computer without a network | [4.12](#412-move-models-to-a-computer-without-a-network), [4.13](#413-install-the-engines-without-a-network), [4.14](#414-sign-and-verify-bundles) |
| Software bill of materials | Writes a CycloneDX SBOM of all that is installed | [4.15](#415-make-a-software-bill-of-materials) |
| Behaviour tests | `llmserver eval` tests a model with built-in probes or with garak | [4.18](#418-test-the-behaviour-of-a-model) |
| Skills for AI agents | Installs checked skills for Claude Code and other agents. Includes the Hugging Face CLI at a fixed version | [3.24](#324-use-the-hugging-face-cli), [3.25](#325-install-skills-for-ai-agents) |
| Programs and scripts | `--json` output for the inspection commands | [3.23](#323-use-llmserver-from-other-programs) |
| Health check and repair | `llmserver doctor` finds problems. `--fix` repairs them | [5.1](#51-do-a-check-of-llmserver) |

---

## Quick start

1. Download the script and make it executable:

   ```bash
   curl -fsSLO https://github.com/pavansreddy/llmserver/blob/main/llmserver
   chmod +x llmserver
   mkdir -p ~/.local/bin && mv llmserver ~/.local/bin/
   ```

2. Do the setup. The setup selects the engines, the network and the context, and installs what is necessary:

   ```bash
   llmserver setup
   ```

3. Start a model:

   ```bash
   llmserver llama:unsloth/Qwen3-8B-GGUF:Q4_K_M
   ```

   Result: llmserver does the security checks, downloads the model, starts it and shows its address.

4. Chat with the model in the terminal:

   ```bash
   llmserver chat
   ```

5. Give the people on your network a web chat:

   ```bash
   llmserver webui
   ```

For each step in detail, see [2. Get started](#2-get-started).

---

## Find a task

| To do this | Read |
|---|---|
| Make sure that the computer is applicable | [2.1](#21-make-sure-that-the-computer-is-applicable) |
| Install llmserver and do the setup | [2.2](#22-install-the-script), [2.3](#23-do-the-setup) |
| Start, stop and chat with a model | [2.4](#24-start-your-first-model), [2.7](#27-stop-the-model), [3.28](#328-chat-with-a-model) |
| Connect a program that uses the OpenAI API | [2.5](#25-connect-a-client), [3.7](#37-connect-clients-to-a-model), [3.33](#333-use-one-address-for-all-models-the-gateway) |
| Know if a model fits on the GPUs | [3.1](#31-examine-the-computer-and-the-models), [3.6](#36-monitor-the-gpus-and-the-models) |
| Operate more than one model | [3.5](#35-start-more-than-one-model), [3.8](#38-decide-what-to-do-when-the-gpus-are-full), [3.9](#39-protect-a-model-from-a-stop) |
| Give a web chat to the people on the network | [3.30](#330-give-people-on-the-network-a-web-chat-open-webui) |
| Ask questions about documents | [3.31](#331-find-answers-in-documents-hybrid-search) |
| Search the web from the chat | [3.32](#332-search-the-web-from-the-chat-searxng) |
| Answer a security warning or approve a model | [4.4](#44-answer-a-security-warning), [4.5](#45-approve-a-model) |
| Install without a network | [3.21](#321-install-through-a-firewall-or-a-mirror), [4.12](#412-move-models-to-a-computer-without-a-network), [4.13](#413-install-the-engines-without-a-network) |
| Find the cause of a problem | [5.1](#51-do-a-check-of-llmserver), [5.3](#53-problems-and-corrective-actions) |
| Find a command, an option or a setting | [6.1](#61-commands), [6.2](#62-options), [6.4](#64-settings) |
| Remove llmserver | [7.11](#711-uninstall-llmserver) |

---

## Contents

- [1. Introduction](#1-introduction)
  - [1.1 What llmserver is](#11-what-llmserver-is)
  - [1.2 What llmserver does](#12-what-llmserver-does)
  - [1.3 Self-contained installation](#13-self-contained-installation)
  - [1.4 Safety of downloads](#14-safety-of-downloads)
  - [1.5 Notices in this manual](#15-notices-in-this-manual)
  - [1.6 Test status](#16-test-status)
- [2. Get started](#2-get-started)
  - [2.1 Make sure that the computer is applicable](#21-make-sure-that-the-computer-is-applicable)
  - [2.2 Install the script](#22-install-the-script)
  - [2.3 Do the setup](#23-do-the-setup)
  - [2.4 Start your first model](#24-start-your-first-model)
  - [2.5 Connect a client](#25-connect-a-client)
  - [2.6 Chat with the model](#26-chat-with-the-model)
  - [2.7 Stop the model](#27-stop-the-model)
- [3. Operate llmserver](#3-operate-llmserver)
  - [3.1 Examine the computer and the models](#31-examine-the-computer-and-the-models)
  - [3.2 Install the engines](#32-install-the-engines)
  - [3.3 Add, remove and change the models in the menu](#33-add-remove-and-change-the-models-in-the-menu)
  - [3.4 Start a model](#34-start-a-model)
  - [3.5 Start more than one model](#35-start-more-than-one-model)
  - [3.6 Monitor the GPUs and the models](#36-monitor-the-gpus-and-the-models)
  - [3.7 Connect clients to a model](#37-connect-clients-to-a-model)
  - [3.8 Decide what to do when the GPUs are full](#38-decide-what-to-do-when-the-gpus-are-full)
  - [3.9 Protect a model from a stop](#39-protect-a-model-from-a-stop)
  - [3.10 Operate one model at a time](#310-operate-one-model-at-a-time)
  - [3.11 Select the GPUs for llmserver](#311-select-the-gpus-for-llmserver)
  - [3.12 Change the settings](#312-change-the-settings)
  - [3.13 Measure the speed](#313-measure-the-speed)
  - [3.14 Use speculative decoding](#314-use-speculative-decoding)
  - [3.15 Download a model before you need it](#315-download-a-model-before-you-need-it)
  - [3.16 Make the models available on the network](#316-make-the-models-available-on-the-network)
  - [3.17 Operate llmserver in a container](#317-operate-llmserver-in-a-container)
  - [3.18 Use an engine that you installed](#318-use-an-engine-that-you-installed)
  - [3.19 Install an engine from a file that you downloaded](#319-install-an-engine-from-a-file-that-you-downloaded)
  - [3.20 Update the engines](#320-update-the-engines)
  - [3.21 Install through a firewall or a mirror](#321-install-through-a-firewall-or-a-mirror)
  - [3.22 Start models automatically after login](#322-start-models-automatically-after-login)
  - [3.23 Use llmserver from other programs](#323-use-llmserver-from-other-programs)
  - [3.24 Use the Hugging Face CLI](#324-use-the-hugging-face-cli)
  - [3.25 Install skills for AI agents](#325-install-skills-for-ai-agents)
  - [3.26 Update the skills](#326-update-the-skills)
  - [3.27 Move the llmserver home](#327-move-the-llmserver-home)
  - [3.28 Chat with a model](#328-chat-with-a-model)
  - [3.29 Use a GGUF file on your computer](#329-use-a-gguf-file-on-your-computer)
  - [3.30 Give people on the network a web chat (Open WebUI)](#330-give-people-on-the-network-a-web-chat-open-webui)
  - [3.31 Find answers in documents (hybrid search)](#331-find-answers-in-documents-hybrid-search)
  - [3.32 Search the web from the chat (SearXNG)](#332-search-the-web-from-the-chat-searxng)
  - [3.33 Use one address for all models (the gateway)](#333-use-one-address-for-all-models-the-gateway)
  - [3.34 Upgrade from a version before 2.1](#334-upgrade-from-a-version-before-21)
- [4. Control the security of models and skills](#4-control-the-security-of-models-and-skills)
  - [4.1 How the security checks work](#41-how-the-security-checks-work)
  - [4.2 Select the security level](#42-select-the-security-level)
  - [4.3 Do a check of a model before you download it](#43-do-a-check-of-a-model-before-you-download-it)
  - [4.4 Answer a security warning](#44-answer-a-security-warning)
  - [4.5 Approve a model](#45-approve-a-model)
  - [4.6 Examine a check record](#46-examine-a-check-record)
  - [4.7 Update a model to a new commit](#47-update-a-model-to-a-new-commit)
  - [4.8 Lock a menu entry to a commit](#48-lock-a-menu-entry-to-a-commit)
  - [4.9 Make the strict level necessary for all users](#49-make-the-strict-level-necessary-for-all-users)
  - [4.10 Use models that you downloaded before version 1.7](#410-use-models-that-you-downloaded-before-version-17)
  - [4.11 Security checks for skills](#411-security-checks-for-skills)
  - [4.12 Move models to a computer without a network](#412-move-models-to-a-computer-without-a-network)
  - [4.13 Install the engines without a network](#413-install-the-engines-without-a-network)
  - [4.14 Sign and verify bundles](#414-sign-and-verify-bundles)
  - [4.15 Make a software bill of materials](#415-make-a-software-bill-of-materials)
  - [4.16 Use local scanners](#416-use-local-scanners)
  - [4.17 Isolate the engines](#417-isolate-the-engines)
  - [4.18 Test the behaviour of a model](#418-test-the-behaviour-of-a-model)
  - [4.19 Limits of the security checks](#419-limits-of-the-security-checks)
- [5. Monitor and correct problems](#5-monitor-and-correct-problems)
  - [5.1 Do a check of llmserver](#51-do-a-check-of-llmserver)
  - [5.2 Recover after a crash of llmserver](#52-recover-after-a-crash-of-llmserver)
  - [5.3 Problems and corrective actions](#53-problems-and-corrective-actions)
- [6. Reference data](#6-reference-data)
  - [6.1 Commands](#61-commands)
  - [6.2 Options](#62-options)
  - [6.3 Model names](#63-model-names)
  - [6.4 Settings](#64-settings)
  - [6.5 Files and folders](#65-files-and-folders)
  - [6.6 Verdicts](#66-verdicts)
  - [6.7 VRAM guard](#67-vram-guard)
  - [6.8 Engine builds](#68-engine-builds)
  - [6.9 Exit status](#69-exit-status)
  - [6.10 Limits](#610-limits)
  - [6.11 Security rules](#611-security-rules)
  - [6.12 Check record](#612-check-record)
  - [6.13 Bundle format](#613-bundle-format)
- [7. Undo, restore and remove](#7-undo-restore-and-remove)
  - [7.1 Stop models](#71-stop-models)
  - [7.2 Remove a pin](#72-remove-a-pin)
  - [7.3 Undo a start that failed](#73-undo-a-start-that-failed)
  - [7.4 Go back to the previous engine build](#74-go-back-to-the-previous-engine-build)
  - [7.5 Go back to the specified versions](#75-go-back-to-the-specified-versions)
  - [7.6 Repair an engine](#76-repair-an-engine)
  - [7.7 Reset the settings](#77-reset-the-settings)
  - [7.8 Withdraw an approval](#78-withdraw-an-approval)
  - [7.9 Delete the check records](#79-delete-the-check-records)
  - [7.10 Remove skills](#710-remove-skills)
  - [7.11 Uninstall llmserver](#711-uninstall-llmserver)
- [Sources](#sources)
- [License](#license)

---

## 1. Introduction

### 1.1 What llmserver is

llmserver is one script for Linux. It operates large language models (LLMs) on your computer.

Each model gets its own network port and its own OpenAI-compatible endpoint: `http://HOST:PORT/v1`. Clients that use the OpenAI API can connect to the endpoint.

llmserver uses one of three engines to operate a model:

| Engine | Model source | Use it for |
|---|---|---|
| llama.cpp | GGUF repositories on Hugging Face | NVIDIA, AMD or CPU. It can put part of a model on the GPU and part on the CPU |
| vLLM | Hugging Face repositories (safetensors, AWQ, GPTQ, FP8) | The highest throughput. It can divide a model across GPUs. It keeps all of the model on the GPU |
| Ollama | The Ollama library, or GGUF repositories through `hf.co/...` | Simple model management. It divides a model between GPU and CPU automatically |

### 1.2 What llmserver does

Before llmserver starts a model, it does these steps:

1. It finds the GPUs and the memory of the computer.
2. It calculates the memory that the model needs.
3. It tells you if the model will operate well, operate slowly, or not operate.
4. It finds the best GPU for the model, next to the models that operate already.
5. If the GPUs are full, it tells you which old models must stop.
6. It does the security checks. It downloads only the files that the engine needs and does a check of each file (see [4](#4-control-the-security-of-models-and-skills)).
7. It starts the model, in a sandbox if bubblewrap is installed, and waits until the model is ready.
8. If the start fails, it starts again the models that it stopped.

### 1.3 Self-contained installation

llmserver keeps all its files in one folder: `~/.llmserver`. This folder is the "llmserver home". It contains:

- uv, a Python installer, and a private Python.
- The engines: llama.cpp, Ollama and vLLM.
- The settings file `llmserver.env`, and the menu, the API key and the tuning rules in `config/`.
- The logs in `logs/`, and the records of the running models in `state/`.
- The skills for AI agents in `skills/`, and the security records in `trust/`.

Only the home folder (mode 700) contains llmserver files. To remove llmserver, use [7.11](#711-uninstall-llmserver). To use a different folder, see [3.27](#327-move-the-llmserver-home).

The model files are not in the home. They stay in the shared caches (`~/.cache/huggingface/hub` and `~/.ollama/models`), so that other programs can use them. To keep the model files in the home, set `LLMSERVER_MODEL_STORE=private`.

llmserver does not change your operating system:

- It does not use sudo.
- It does not install system packages.
- It does not use the system Python or your Python.
- It does not change `PATH` or your shell files.
- It does not install system services.

From the operating system, llmserver needs only the GPU kernel driver. The Vulkan build of llama.cpp also needs the Vulkan driver of the graphics system.

### 1.4 Safety of downloads

- Each file that llmserver downloads has a fixed version and a fixed SHA-256 checksum in the script.
- llmserver does a check of each file before it uses the file. If the checksum is not correct, llmserver deletes the file. It does not install it.
- llmserver uses HTTPS only.
- Python packages come from lock files that contain a checksum for each file.
- llmserver records a checksum of each engine after installation. Then it can find changes to the engine later.
- Before llmserver downloads a model, it reads the Hugging Face scan results of each file. It downloads only the files that the engine needs, at a fixed commit, and does a check of each file. See [4](#4-control-the-security-of-models-and-skills).

### 1.5 Notices in this manual

> **CAUTION:** A caution tells you about a risk of damage to data, to models or to a service that operates.

> **NOTE:** A note gives you more information about a step.

### 1.6 Test status

The self-contained installation was tested on Linux x86-64 without a GPU:
- the runtime;
- llama.cpp (CPU and Vulkan builds) and Ollama;
- operation of real models;
- upgrade, rollback and uninstall.

The security checks were tested with real Hugging Face models and a real llama.cpp, and with a simulated Hugging Face for the refusals. The skills were tested with the real Hugging Face skills marketplace, and with a local copy for the refusals. The GitHub issue check was tested with a simulated GitHub only.

Export, import and signatures were tested between two separate homes. `install --from` was tested with the network blocked, for the runtime and llama.cpp. The vLLM part of a bundle was tested up to the package list: all 198 locked packages were found by checksum. The local scanners were tested with stand-in programs.

The sandbox was tested with a real llama.cpp: the engine could not read the home folder or write to the system or the model cache. `eval` was tested with a real llama.cpp model, the built-in probes and a real garak run.

The single-folder layout was tested with an upgrade from version 2.0.0 (with a running model, a skill and a real helper runtime), a moved home, and an uninstall with and without `--keep-config`.

The menu, settings, key and autostart commands were tested with a terminal (for the questions) and without one. `autostart` was tested with a simulated `systemctl` only. `chat` was tested with a real llama.cpp model, and with a simulated server for the errors. Open WebUI 0.12.3 was installed with `llmserver install webui` and tested in the sandbox: account creation, the model list through the gateway, and a chat with a real llama.cpp model. The gateway on the network was tested with and without the API key. Hybrid document search was tested with the real Open WebUI and the real fast reranker. The best reranker was tested with the real bge-reranker-v2-m3 file on a real llama.cpp (CPU): a question in English found the answer in a French passage first. SearXNG was installed from the real source (with git; the archive host was blocked on the test network) and answered real searches. Open WebUI got the search results; the test network could not resolve the result pages, so the reading of web pages was not tested. A local GGUF file was tested with a real llama.cpp that was unpacked into a folder of the user (`llama_source`), and the installation from a downloaded llama.cpp release file was tested with the real file.

The vLLM installation method was tested on a small package only. GPU operation was tested with simulated GPUs only. Before your first real start, use `llmserver check`, `llmserver capacity` and `--dry-run`.

---

## 2. Get started

### 2.1 Make sure that the computer is applicable

Make sure that the computer agrees with these requirements:

| Item | Requirement |
|---|---|
| Operating system | Linux, x86-64 or ARM64. WSL2 and containers are also applicable. Other systems are not applicable |
| Shell | bash 4.4 or newer |
| Programs | curl, tar, gzip, GNU coreutils, findutils, grep, sed and awk |
| Recommended programs | `ss` or `lsof`, `flock`, `setsid`, `timeout`. `numactl` or `taskset` for NUMA computers. `pgrep` for `doctor` |
| C library | glibc 2.35 or newer. glibc 2.39 or newer for the CUDA builds of llama.cpp and for vLLM on AMD |
| GPU driver | NVIDIA: the NVIDIA driver with `nvidia-smi`. AMD: the `amdgpu` kernel driver. vLLM and Ollama on AMD also need `amdkfd` |
| Disk | About 200 MB for the runtime, 25 to 800 MB for llama.cpp, 1.5 to 2.6 GB for Ollama, 10.5 to 14 GB for vLLM (NVIDIA includes 1.5 GB of FlashInfer kernels). Optional: 2.5 GB for Open WebUI with its document models, 150 MB for SearXNG, 0.6 GB for the best reranker. Add the size of each model plus 10% |
| Network | HTTPS access to `github.com`, `pypi.org`, `files.pythonhosted.org`, `download.pytorch.org`, `wheels.vllm.ai`, `huggingface.co` and `registry.ollama.ai`. Web search also needs access to the public search engines. You can use mirrors (see [3.21](#321-install-through-a-firewall-or-a-mirror)) |

> **NOTE:** You do not need Python on the computer. llmserver installs its own Python.

### 2.2 Install the script

1. Make the script executable:

   ```bash
   chmod +x llmserver
   ```

2. Move the script to a folder in your `PATH`:

   ```bash
   mkdir -p ~/.local/bin && mv llmserver ~/.local/bin/
   ```

3. Do a check of the version:

   ```bash
   llmserver --version
   ```

   Result: The command shows `llmserver 2.7.1`.

### 2.3 Do the setup

1. Start the setup:

   ```bash
   llmserver setup
   ```

   Result: The setup shows the GPUs and the memory that it found.

2. Answer each question. Push Enter to accept the value in brackets.

   | Question | What to type |
   |---|---|
   | Engines to enable | The numbers of the engines, for example `1,3`. The setup selects the applicable engines for you |
   | Default engine | The engine for Hugging Face repositories that more than one engine can operate |
   | Ollama mode | Only if Ollama operates already on port 11434. Type `1` to use it, or `2` for separate Ollama servers |
   | Network | `1` for this computer only. `2` for other computers and containers. `3` for one address |
   | First port | The first network port. Each model gets the next free port |
   | Authentication | Only for network access. Type `1` for an API key |
   | Context | The context size of each model in tokens. The default is 32768 |
   | Most models at once | `0` for as many models as the GPUs can hold. `1` for one model at a time |
   | Eviction | What to do when a new model does not fit. See [3.8](#38-decide-what-to-do-when-the-gpus-are-full) |
   | Model | One model name on each line. Push Enter on an empty line to continue |

3. Read the installation plan. The plan shows each component and its size.

4. Type `y` to install.

   Result: llmserver installs the components into the llmserver home. The setup writes the settings file `~/.llmserver/llmserver.env` and the menu `~/.llmserver/config/models`.

> **NOTE:** You can do the setup again at any time. The setup does not delete your menu. It keeps your engine variables in `~/.llmserver/llmserver.env`.

> **NOTE:** At the end, the setup asks if it must install Open WebUI, a web chat for people on your network, who can use it, and if it must also search the web. See [3.30](#330-give-people-on-the-network-a-web-chat-open-webui).

### 2.4 Start your first model

1. Do a check of the model before you download it:

   ```bash
   llmserver check llama:ORG/REPO-GGUF:QUANT
   ```

   Result: The command shows a verdict (see [6.6](#66-verdicts)).

2. Start the model:

   ```bash
   llmserver llama:ORG/REPO-GGUF:QUANT
   ```

   Result: llmserver downloads the model, starts it and shows `ready` with the address.

3. Make sure that the model operates:

   ```bash
   llmserver ps
   ```

   Result: The model shows with its port and the state `listening`.

### 2.5 Connect a client

1. Load the address and the key into your shell:

   ```bash
   eval "$(llmserver env 1)"
   ```

2. Send a request:

   ```bash
   curl -s "$OPENAI_BASE_URL/chat/completions" \
     -H "Authorization: Bearer $OPENAI_API_KEY" -H 'Content-Type: application/json' \
     -d '{"model":"'"$LLMSERVER_MODEL"'","messages":[{"role":"user","content":"hello"}]}'
   ```

   Result: The model sends a reply.

### 2.6 Chat with the model

1. Start a chat:

   ```bash
   llmserver chat
   ```

   Result: llmserver connects to the model that operates. If no model operates, it shows the menu and asks which model to start.

2. Type a message and push Enter. The answer shows while the model makes it. After each answer, llmserver shows the number of tokens, the speed and the time to the first token. For a reasoning model, these include the thinking tokens.

3. Type `/exit` (or push Ctrl-D) to stop the chat.

See [3.28](#328-chat-with-a-model) for more.

### 2.7 Stop the model

1. Stop the model:

   ```bash
   llmserver stop 1
   ```

   Result: The command shows `stopped 1 model(s)`.

---

## 3. Operate llmserver

### 3.1 Examine the computer and the models

1. Show the hardware, the engines and a verdict for each menu entry:

   ```bash
   llmserver check
   ```

2. Show the details for one model:

   ```bash
   llmserver check MODEL
   ```

   Result: The command shows:
   - the verdict and the memory that the model needs;
   - the context;
   - an estimate of the speed (after [3.13](#313-measure-the-speed));
   - recommendations, for example a smaller quantization.

3. Show the verdict at a different context size:

   ```bash
   llmserver check MODEL --ctx 65536
   ```

> **NOTE:** The `check` command does not download the model. It reads the size from the registry.

### 3.2 Install the engines

1. Show the installation plan:

   ```bash
   llmserver install all -n
   ```

2. Install the engines that you need:

   ```bash
   llmserver install llama ollama
   ```

3. Type `y` to install.

4. Do a check of the installation:

   ```bash
   llmserver versions
   ```

   Result: Each component shows with its version and its build.

> **NOTE:** llmserver selects the build for your GPUs (see [6.8](#68-engine-builds)). To select a different llama.cpp build, set `LLMSERVER_LLAMA_BUILD` in `~/.llmserver/llmserver.env`.

> **NOTE:** If you start a model and its engine is not installed, llmserver asks you to install it.

### 3.3 Add, remove and change the models in the menu

1. Add a model:

   ```bash
   llmserver model add vllm:Qwen/Qwen3-8B
   ```

   Result: llmserver makes sure that the model exists on Hugging Face. Then it adds the model to the menu. If the name is not correct, llmserver adds nothing and shows similar names ("did you mean ...?").

2. To add engine flags to the entry, put them after `--`:

   ```bash
   llmserver model add llama:ORG/REPO-GGUF:QUANT -- -c 65536 --parallel 2
   ```

3. Show the menu:

   ```bash
   llmserver model list
   ```

4. Remove an entry:

   ```bash
   llmserver model rm
   ```

   Result: llmserver shows the numbered menu and asks which entries to remove. Type one or more numbers (for example `2 4`, or `2-4`), then Enter. Push Enter without a number to remove nothing.

   You can also give the entries directly: `llmserver model rm 4` or `llmserver model rm vllm:Qwen/Qwen3-8B`.

5. To also delete the files that llmserver downloaded for the entry, add `--purge`:

   ```bash
   llmserver model rm 4 --purge
   ```

   Result: llmserver asks before it deletes the files. It deletes only files that it downloaded and that no other menu entry uses. It also deletes the check records of the model. It does not delete a model that operates; stop it first.

6. To delete the downloaded files of a model later, use `llmserver model purge MODEL`.

7. To change the menu in a text editor, type `llmserver model edit`.

   Result: The menu opens in your editor (`$VISUAL` or `$EDITOR`; else `nano` or `vi`). When you close the editor, llmserver shows each line that is not correct.

The menu is the file `~/.llmserver/config/models`. Each line is one model, with optional words after the model:

```
llama:ORG/REPO-GGUF:QUANT
llama:ORG/OTHER-GGUF:QUANT   -c 65536 --parallel 2
vllm:ORG/REPO                port=9000
ollama:NAME:TAG              OLLAMA_NUM_PARALLEL=2
```

> **NOTE:** These words after a model are for llmserver, not for the engine: `port=N`, `draft=MODEL` and `draft-n=N`.

> **NOTE:** When you start a model by its name, llmserver adds it to the menu.

> **NOTE:** A name has the form `[engine:]owner/model`, for example `Qwen/Qwen3-8B` or `vllm:Qwen/Qwen3-8B`. If you do not write an engine, llmserver selects one. A colon without an engine (`:Qwen/Qwen3-8B`) is the same as no engine.

### 3.4 Start a model

1. Start a model by its menu number or its name:

   ```bash
   llmserver 2
   ```

2. Use these options if necessary:

   | Option | Result |
   |---|---|
   | `--port N` | Uses port N |
   | `--ctx N` | Uses a context of N tokens |
   | `--ctx fit` | Uses the largest context that fits in the free GPU memory |
   | `--foreground` | Shows the output of the server. Ctrl-C stops the model |
   | `-n` | Shows what llmserver will do. It changes nothing |
   | `-- FLAGS` | Sends the flags after `--` to the engine |

3. To start a model again with new settings, use `--restart`:

   ```bash
   llmserver 2 --restart --ctx 65536
   ```

   Result: The model starts again on the same port.

> **NOTE:** A model keeps the same port each time that it starts.

### 3.5 Start more than one model

1. Start the first model:

   ```bash
   llmserver 1
   ```

2. Start the second model:

   ```bash
   llmserver 2
   ```

   Result: The second model gets the next free port. llmserver puts it on the GPU that has space for it.

3. Show all models:

   ```bash
   llmserver ps
   ```

### 3.6 Monitor the GPUs and the models

1. Show the memory of each GPU and the models that use it:

   ```bash
   llmserver capacity
   ```

   Result: The command shows:
   - each GPU, its free memory, and the models and other programs on it;
   - each menu entry: "fits now", "fits after stopping" other models, or "never fits";
   - the number of menu models that can operate at the same time.

2. Show the output of one model:

   ```bash
   llmserver logs 2
   ```

3. Show the activity of llmserver:

   ```bash
   llmserver log
   ```

4. Show only the warnings and the errors:

   ```bash
   llmserver log --errors
   ```

### 3.7 Connect clients to a model

1. Show the variables for a model:

   ```bash
   llmserver env 2
   ```

2. Copy the variables into the configuration of the client.

> **NOTE:** If more than one model operates and you do not give a model, `env` shows the address of each model.

### 3.8 Decide what to do when the GPUs are full

1. Set `evict` to one of the values in the table, for example:

   ```bash
   llmserver config set evict lru
   ```

   | Value | In a terminal | Without a terminal |
   |---|---|---|
   | `ask` (default) | llmserver shows the models that must stop and asks you | llmserver stops nothing and refuses the start |
   | `lru` | llmserver stops the least recently used models | The same |
   | `never` | llmserver refuses the start | The same |

2. To override the setting for one start, use one of these options:

   | Option | Result |
   |---|---|
   | `--evict` | Stops the least recently used models without a question |
   | `--offload` | Starts the model partially on the CPU. It stops no other model |

> **CAUTION:** When llmserver stops a model, clients of that model lose their connection. Pin the models that must not stop (see [3.9](#39-protect-a-model-from-a-stop)).

### 3.9 Protect a model from a stop

1. Pin the model:

   ```bash
   llmserver pin 1
   ```

   Result: llmserver does not stop this model to make space for other models.

> **NOTE:** You can stop a pinned model with `llmserver stop`. To remove the pin, see [7.2](#72-remove-a-pin).

### 3.10 Operate one model at a time

1. Set these values:

   ```bash
   llmserver config set max_models 1
   llmserver config set evict lru
   ```

   Result: When you start a model, llmserver stops the model that operates.

### 3.11 Select the GPUs for llmserver

1. Show the GPU numbers:

   ```bash
   llmserver check
   ```

2. Set the GPU numbers, for example:

   ```bash
   llmserver config set gpus 0,2
   ```

> **NOTE:** llmserver also obeys `CUDA_VISIBLE_DEVICES`, `ROCR_VISIBLE_DEVICES` and `HIP_VISIBLE_DEVICES` from its environment. It uses only the GPUs that these variables permit.

### 3.12 Change the settings

1. Show the settings and where each value comes from:

   ```bash
   llmserver config
   ```

2. Change a setting. You can write the name without `LLMSERVER_` and in lowercase:

   ```bash
   llmserver config set port 9000
   ```

   Result: llmserver does a check of the value before it saves it. If the value is not correct, llmserver saves nothing and tells you why. If the name is not correct, llmserver shows the nearest name.

3. Set an engine variable in capitals, for example a token for gated models:

   ```bash
   llmserver config set HF_TOKEN hf_...
   ```

   Result: llmserver gives the variable to every engine that it starts. `llmserver config` shows only the start of a token, key, secret or password.

4. To go back to the default value, remove the setting:

   ```bash
   llmserver config unset port
   ```

5. To change many settings, open the file in your editor:

   ```bash
   llmserver config edit
   ```

   Result: When you close the editor, llmserver does a check of the settings and shows each problem.

> **NOTE:** If a setting is not correct, the other commands stop with an error. `llmserver config set`, `unset` and `edit` still operate, so that you can correct the setting.

> **NOTE:** llmserver reads the settings in this sequence. A later source overrides an earlier source:
> 1. `/etc/llmserver_env`;
> 2. `~/.llmserver/llmserver.env`;
> 3. variables in your environment;
> 4. command options.
>
> If a variable in your environment overrides the file, `llmserver config set` tells you.

> **NOTE:** The settings file has mode 600 (only you can read it). `LLMSERVER_HOME` cannot be in this file, because the file is in the home (see [3.27](#327-move-the-llmserver-home)).

### 3.13 Measure the speed

1. Start a model that fits fully on the GPU.

2. Measure the model:

   ```bash
   llmserver bench 1
   ```

   Result: llmserver measures a short prompt and a long prompt (each several times, after a warm-up), and then asks one real question. It shows these values:

   | Value | Meaning |
   |---|---|
   | TTFT | Time to the first token. Most of it is the prefill: the model reads the prompt |
   | prompt speed | Prompt tokens per second during the prefill (long prompt only) |
   | tokens generated, generation time | The decode: the model writes exactly 128 tokens (`--gen N` changes this) |
   | decode speed | Generated tokens per second, after the first token |
   | TPOT | Time per output token after the first one: the same as the decode speed, in milliseconds |
   | ITL p50, p90, p99 | The gaps between tokens as they arrive: typical (p50), and the slowest 10% and 1%. A large p99 shows stutter |
   | E2E latency | From the request to the last token |

   For the real question, llmserver shows the thinking (tokens and time), the time to the first answer token, the answer (tokens and time) and the E2E latency. A model that does not think answers directly.

   llmserver keeps the result.

   Add `--json` to get all values for a program. Add `--reasoning low` (or `medium`, `high`, `none`) to measure the real question at that reasoning level.

3. Compare the results:

   ```bash
   llmserver bench --history
   ```

> **NOTE:** After one measurement, `check` and `list` show an estimate of the speed for other models.

> **NOTE:** The short and long measurements use a fixed number of tokens, so they compare well between runs and models. Only the real question shows how much a model thinks. For a reasoning model, thinking tokens in the fixed measurements are counted as generated tokens. If the server has no plain completion endpoint, llmserver measures through the chat endpoint.

> **NOTE:** The long prompt must fit in the context of the server. With `--ctx auto`, llmserver asks the server for its context and makes the prompt shorter if necessary.

> **NOTE:** If Ollama keeps part of the model in system RAM (because the GPUs are full), `bench` tells you, and `llmserver ps` shows `N%-on-cpu`. Generation is then much slower than the GPUs permit. Stop the other models and start the model again to measure it on the GPUs only.

### 3.14 Use speculative decoding

1. Start a model with a small draft model of the same family:

   ```bash
   llmserver llama:ORG/REPO-GGUF:QUANT --draft llama:ORG/SMALL-GGUF:QUANT
   ```

2. For vLLM, you can use prompt lookup instead of a draft model:

   ```bash
   llmserver vllm:ORG/REPO --draft ngram
   ```

3. Measure the speed with the draft and without the draft (`--no-draft`). Compare the results.

> **NOTE:** Ollama does not support speculative decoding.

### 3.15 Download a model before you need it

1. Download the model:

   ```bash
   llmserver download MODEL
   ```

   Result: llmserver does the security checks, downloads the model and does a check of each file. The models that operate do not stop.

### 3.16 Make the models available on the network

1. Set the address:

   ```bash
   llmserver config set host 0.0.0.0
   ```

2. Start the model again with `--restart`.

3. Give the API key to the clients. `llmserver env MODEL` shows it as `OPENAI_API_KEY`. `llmserver key show` shows where it is.

4. Open the ports in the firewall.

5. To make a new key (for example, if the key was disclosed), type `llmserver key new`. Then start the models again with `--restart`, and give the new key to the clients. llmserver shows the models that still use the old key.

> **CAUTION:** Ollama cannot check an API key. llmserver refuses to put Ollama on the network. To permit it, type `llmserver config set allow_insecure 1`. Use this only on a trusted network.

> **CAUTION:** The API key does not encrypt the traffic. On a network that is not trusted, use a TLS proxy.

### 3.17 Operate llmserver in a container

1. Give the container access to the GPUs. For NVIDIA, use the NVIDIA container toolkit. For AMD, pass `/dev/kfd` and `/dev/dri`.

2. Mount `/sys` into the container. Read-only access is sufficient.

3. Start the container, for example:

   ```bash
   docker run --gpus all --device /dev/kfd --device /dev/dri -v /sys:/sys:ro ...
   ```

4. Do a check of the limits:

   ```bash
   llmserver check
   ```

   Result: The `Limits` line shows the memory limit, the CPU limit and the GPU masks of the container.

> **NOTE:** llmserver uses the memory limit of the cgroup, not the memory of the host. It does not use a GPU that the container cannot open.

### 3.18 Use an engine that you installed

1. Tell llmserver where the engine is. Use one of these values:

   | Value | Engine that llmserver uses |
   |---|---|
   | `system` | The program in your `PATH` (`llama-server` or `llama`, `vllm`, `ollama`) |
   | A folder, for example `/opt/llama.cpp/build/bin` | The program in that folder. llmserver also looks in `bin/` and `build/bin/` |
   | A program, for example `/opt/llama.cpp/build/bin/llama-server` | That program |

   For example:

   ```bash
   llmserver config set llama_source /opt/llama.cpp/build/bin
   ```

   Result: llmserver does a check that the path exists. `llmserver versions` shows the program that llmserver uses.

2. To go back to the copy of llmserver, type `llmserver config set llama_source managed`.

> **NOTE:** llmserver does not change or delete an engine that you installed. In the sandbox, the engine sees its folder read-only.

> **NOTE:** You can also use `ollama_source` and `vllm_source` (for vLLM, the folder of its Python environment).

### 3.19 Install an engine from a file that you downloaded

If you downloaded the llama.cpp or Ollama release file yourself (for example, on a computer with a slow or limited network):

1. Install from the file:

   ```bash
   llmserver install llama --from ~/Downloads/llama-b11798-bin-ubuntu-x64.tar.gz
   ```

   Result: llmserver compares the SHA-256 of the file with the value that it specifies for this release. If they are the same, llmserver installs the file as its own copy. Upgrade, rollback and `doctor` operate as usual.

2. If llmserver refuses the file because it is a different release, unpack the file and use it as your own engine (see [3.18](#318-use-an-engine-that-you-installed)).

> **NOTE:** The file name tells llmserver which build it is (for example CPU, Vulkan or CUDA). For a CUDA build, put the `cudart-...` file in the same folder; else llmserver downloads it. For Ollama with AMD GPUs, put both `ollama-linux-amd64.tar.zst` and `ollama-linux-amd64-rocm.tar.zst` in the same folder and give the first one.

> **NOTE:** The runtime (uv, Python and the helper packages) still comes from the network. For a computer without a network, use a bundle (see [4.13](#413-install-the-engines-without-a-network)).

### 3.20 Update the engines

1. Install the versions that this llmserver release specifies:

   ```bash
   llmserver upgrade
   ```

2. To install the newest version of an engine, use `--latest`:

   ```bash
   llmserver upgrade llama --latest
   ```

   Result: llmserver installs the newest release only if the release publishes a SHA-256 checksum for each file.

3. Do a check of the versions:

   ```bash
   llmserver versions
   ```

> **NOTE:** Models that operate keep the old engine until you start them again.

> **NOTE:** To go back to the old engine, see [7.4](#74-go-back-to-the-previous-engine-build).

### 3.21 Install through a firewall or a mirror

1. Set the mirrors that you need with `llmserver config set NAME VALUE`, for example `llmserver config set UV_INDEX_URL https://mirror.example/simple`:

   | Setting | Use |
   |---|---|
   | `LLMSERVER_GITHUB` | A mirror for GitHub release files |
   | `UV_INDEX_URL` | A mirror for Python packages |
   | `UV_PYTHON_INSTALL_MIRROR` | A mirror for Python builds |
   | `HTTPS_PROXY`, `SSL_CERT_FILE` | A proxy and its certificate |
   | `UV_TORCH_BACKEND` | The PyTorch build for vLLM, for example `cu130` |

> **NOTE:** llmserver does a check of each file against its checksum. A mirror cannot change the installed files.

> **NOTE:** For a computer without network access:
> 1. Copy `~/.llmserver` from a computer with the same architecture. A different path is permitted: llmserver adjusts the folder at the next start.
> 2. Type `llmserver config set auto_install off`.

### 3.22 Start models automatically after login

1. Do the setup first (see [2.3](#23-do-the-setup)).

2. Enable the models, for example menu entries 1 and 2:

   ```bash
   llmserver autostart on 1 2
   ```

   Result: llmserver writes the systemd user service `~/.config/systemd/user/llmserver@.service` and enables one instance for each model. The models start when you log in.

3. To start them now, type the command that llmserver shows, for example `systemctl --user start llmserver@...service`.

4. To start the models when the computer starts, without a login:

   ```bash
   loginctl enable-linger $USER
   ```

5. Show the models that start automatically:

   ```bash
   llmserver autostart
   ```

6. To stop the automatic start of a model:

   ```bash
   llmserver autostart off 1
   ```

   Result: When no model is left, llmserver deletes the service file.

> **NOTE:** The service uses the model name, not the menu number. If you change the menu, the service continues to start the same model.

> **NOTE:** This needs systemd. The service file was tested with a simulated `systemctl` only.

### 3.23 Use llmserver from other programs

1. Add `--json` to an inspection command:

   ```bash
   llmserver capacity --json
   ```

   Result: The command writes one JSON object to standard output. Messages go to standard error.

2. Read the values that you need, for example:

   ```bash
   llmserver capacity --json | jq '.at_once.count'
   ```

> **NOTE:** These commands accept `--json`: `ps`, `check`, `capacity`, `list`, `doctor`, `versions`, `trust`, `models` and `skills`. Sizes are in MiB and times are in seconds.

### 3.24 Use the Hugging Face CLI

llmserver installs a fixed version of the Hugging Face CLI (`hf`) in its runtime. It does not put `hf` on `PATH`.

1. Use the CLI through llmserver:

   ```bash
   llmserver hf auth login
   ```

   Result: The token goes to the usual Hugging Face location. The engines use it for gated models.

2. To use the name `hf` in your shell, make a link:

   ```bash
   llmserver hf --link
   ```

   Result: llmserver writes `~/.local/bin/hf`. This file starts `llmserver hf`.

3. To remove the link, type `llmserver hf --unlink`.

> **NOTE:** llmserver refuses `hf update`, `hf extensions install`, `hf skills add` and `hf skills update`. Use `llmserver upgrade runtime` and `llmserver skills` instead.

> **NOTE:** `hf --link` does not replace an `hf` file that llmserver did not write.

### 3.25 Install skills for AI agents

A skill tells an AI agent (for example Claude Code) how to do a task. llmserver keeps each skill in `~/.llmserver/skills` and makes a link to it in `~/.agents/skills` and in `~/.claude/skills`. The agents find the skill through the links.

There are two types of skills:

| Type | Skills | Source |
|---|---|---|
| Generated | `llmserver`, `hf-cli` | llmserver writes them. `hf-cli` comes from the fixed `hf` and uses `llmserver hf` |
| Marketplace | For example `hf-mem`, `huggingface-papers` | The Hugging Face skills marketplace only |

1. Show the skills:

   ```bash
   llmserver skills list
   ```

2. Install the generated skills:

   ```bash
   llmserver skills add
   ```

3. To install a marketplace skill, do a check first:

   ```bash
   llmserver skills check NAME
   ```

4. Read the skill:

   ```bash
   llmserver skills preview NAME
   ```

5. Install the skill:

   ```bash
   llmserver skills add NAME
   ```

   Result: llmserver shows the files and the findings. It asks before it installs a skill with warnings.

6. To install a skill for the current project only, add `--project`.

> **CAUTION:** An agent follows the instructions in a skill, and it can run the scripts of a skill. Read a skill before you install it.

> **NOTE:** llmserver never runs the scripts of a skill. It installs them without execute permission.

> **NOTE:** `LLMSERVER_SKILLS=ask` (the default) makes `setup` offer the generated skills one time.

### 3.26 Update the skills

1. Update all skills that llmserver installed:

   ```bash
   llmserver skills update
   ```

   Result: llmserver does the checks again and shows the changed files.

2. If you changed a skill, llmserver does not update it. To replace your changes, add `--force`.

> **NOTE:** `llmserver upgrade runtime` and `llmserver doctor --fix` update the generated skills automatically.

### 3.27 Move the llmserver home

1. Stop all models:

   ```bash
   llmserver stop all
   ```

2. Move the folder, for example:

   ```bash
   mv ~/.llmserver /data/llmserver
   ```

3. Set `LLMSERVER_HOME` in your shell profile (or, for all users, in `/etc/llmserver_env`):

   ```bash
   export LLMSERVER_HOME=/data/llmserver
   ```

   > **NOTE:** Do not set `LLMSERVER_HOME` in `llmserver.env`. That file is in the home, so llmserver cannot read it before it knows the home. llmserver shows a warning and ignores the setting.

4. Run a command, for example `llmserver list`.

   Result: llmserver finds that the folder moved. It adjusts the Python environments, the links in the folder and the links to the skills. It shows how many items it adjusted.

### 3.28 Chat with a model

1. Start a chat with a specified model (a menu number, a name or a port):

   ```bash
   llmserver chat 2
   ```

   Result: If the model does not operate, llmserver asks if it must start it. If more than one model operates and you do not specify one, llmserver asks which.

2. Type messages. The model sees all of the conversation. Use these commands in the chat:

   | Command | Result |
   |---|---|
   | `/help` | Shows the commands |
   | `/think` | Shows or hides the thinking of a reasoning model |
   | `/reasoning LEVEL` | Sets how much a reasoning model thinks: `low`, `medium`, `high`, `none` (no thinking) or `off` (the default of the model). `/reasoning` alone shows it |
   | `/system TEXT` | Sets the system prompt, from the next message. `/system` alone shows it |
   | `/clear` | Forgets the conversation. Keeps the system prompt |
   | `/retry` | Asks for the last answer again |
   | `/save FILE` | Writes the conversation to FILE as Markdown |
   | `"""` | Starts and ends a message with more than one line |
   | `/exit` or Ctrl-D | Stops the chat |
   | Ctrl-C | Stops the answer that the model writes now |

3. To set a system prompt from the start, add `--system`:

   ```bash
   llmserver chat 2 --system "Answer in one sentence."
   ```

   To set how much a reasoning model thinks, add `--reasoning low`, `medium`, `high` or `none`:

   ```bash
   llmserver chat 2 --reasoning low
   ```

   llmserver sends the level as `reasoning_effort` with each request (for llama.cpp and vLLM, also to the chat template). This has a stronger effect than a sentence such as "Reasoning: low" in the system prompt. A model or engine that does not support it ignores it.

4. To ask one question and get only the answer (for example, in a script), give the message after the model, or send it through a pipe:

   ```bash
   llmserver chat 2 "What is the capital of France?"
   cat notes.txt | llmserver chat 2
   ```

   Result: llmserver writes only the answer to standard output, and stops. Without a terminal, llmserver does not start a model unless you add `--yes`.

> **NOTE:** If the model thinks before it answers (reasoning models), llmserver shows `thinking...` and then how long the model thought. To see the thinking, type `/think` in the chat, or start the chat with `--think`. An answer to one question (a message or a pipe) never contains the thinking.

> **NOTE:** If the conversation becomes longer than the context of the model, llmserver tells you. Type `/clear`, or start the model again with a larger `--ctx`.

> **NOTE:** llmserver keeps the lines that you typed (not the answers) in `~/.llmserver/state/chat_history` (mode 600), so that the up arrow shows earlier messages. It does not keep conversations unless you use `/save`.

### 3.29 Use a GGUF file on your computer

1. Add the file to the menu:

   ```bash
   llmserver model add ~/models/Winnow-12B-Q8_0.gguf
   ```

   Result: llmserver makes sure that the file exists and is a GGUF file. The menu shows the entry as `llama:~/models/Winnow-12B-Q8_0.gguf`.

2. Start it (or chat with it):

   ```bash
   llmserver 3
   ```

   Result: the first time, llmserver tells you that the file is not from Hugging Face and asks if it can continue (or add `--yes`). Then it does these checks:
   - the file must be a GGUF file and not a program;
   - it calculates and records the SHA-256 of the file (a large file takes about one minute);
   - it runs the local scanners, if they are installed (see [4.16](#416-use-local-scanners)).

3. Clients use the file name without `.gguf` as the model name, for example `Winnow-12B-Q8_0`.

> **NOTE:** llmserver uses the file where it is. It does not copy, move or delete it: `model rm --purge`, `model purge` and `uninstall` keep it.

> **NOTE:** At each start, llmserver makes sure that the file did not change (size and modification time). If the file changed, llmserver does the checks again and asks again. `llmserver doctor --deep` also compares the SHA-256.

> **NOTE:** A model that is split into parts (`...-00001-of-00003.gguf`) is added with its first part. All parts must be in the same folder. If the folder has a vision projector (`mmproj-*.gguf`), llmserver gives it to llama.cpp.

> **NOTE:** A GGUF file is served by llama.cpp. The path must not contain spaces or `:`.

> **NOTE:** With `LLMSERVER_SECURITY=strict`, a local file needs `llmserver trust approve PATH`, like every model.

> **NOTE:** If the model is on Hugging Face, you can let llmserver download only the file that you need, for example `llmserver model add llama:EldanRing/Winnow-12B:Q8_0`. Then llmserver also has the scan results of Hugging Face.

### 3.30 Give people on the network a web chat (Open WebUI)

Open WebUI is a web chat. People open it in a browser, log in, select one of the running models and chat. llmserver installs and operates it.

1. Start it:

   ```bash
   llmserver webui
   ```

   Result: The first time, llmserver asks to install Open WebUI 0.12.3 (about 2.5 GB, with CPU PyTorch) into `~/.llmserver/tools`. Then it starts the gateway (see [3.33](#333-use-one-address-for-all-models-the-gateway)) and Open WebUI, and shows the address, for example `http://192.168.1.20:3000`.

2. Open the address in a browser and create the first account. The first account is the administrator.

3. Give the address to the other people. To let them in, do one of these steps in Open WebUI as the administrator:
   - Add their accounts in **Admin Panel > Users**.
   - Turn on new sign-ups in **Admin Panel > Settings**. New accounts then wait until you approve them in **Admin Panel > Users**.

4. Start the models that the people can use, for example `llmserver 1`. Open WebUI shows every running model. A model that you start later shows immediately. A model that you stop disappears.

5. Show the state, or stop Open WebUI and the services that it uses:

   ```bash
   llmserver webui status
   llmserver webui stop
   ```

6. After you change a `webui_*` setting, apply it:

   ```bash
   llmserver webui restart
   ```

To start Open WebUI when you log in, type `llmserver autostart on webui`.

| Setting | Default | Function |
|---|---|---|
| `LLMSERVER_WEBUI_HOST` | `0.0.0.0` | `0.0.0.0`: people on the network can use it. `127.0.0.1`: only this computer |
| `LLMSERVER_WEBUI_PORT` | `3000` | The port of Open WebUI |
| `LLMSERVER_WEBUI_AUTH` | `on` | `off` (no login) is permitted only with `LLMSERVER_WEBUI_HOST=127.0.0.1` |
| `LLMSERVER_WEBUI` | `ask` | `ask`: the setup asks if it must install Open WebUI. `on`: the setup installs it. `off`: the setup does not ask |

llmserver sets up Open WebUI for safe use on a network:
- Each person needs an account. New sign-ups are off after the first account, or wait for approval.
- Its only connection is the gateway. It cannot connect to the models directly or to Ollama.
- It does not download at run time (offline mode). Its embedding model and its reranker (for documents in chats) are downloaded once, at a fixed commit, and llmserver compares the SHA-256 of each file. They cannot run code from their files.
- Community sharing and the check for new versions are off.
- It operates in the sandbox (see [4.17](#417-isolate-the-engines)). It can write only to its data folder, `~/.llmserver/webui`.
- It gets a private secret key, and none of your tokens (for example `HF_TOKEN`).
- It accepts forwarded client addresses (`X-Forwarded-For`) only from this computer, so a client cannot hide its address.
- It cannot fetch web pages on this computer or on your private network, for example when a person gives it a link.

Documents and web search: see [3.31](#331-find-answers-in-documents-hybrid-search) and [3.32](#332-search-the-web-from-the-chat-searxng).

> **NOTE:** The accounts, chats and uploads are in `~/.llmserver/webui/data`. `llmserver uninstall --keep-config` keeps them.

> **NOTE:** Open WebUI has its own license. You must keep its "Open WebUI" branding if more than 50 people use it in 30 days, unless you have an enterprise license.

> **NOTE:** The traffic between the browsers and Open WebUI is not encrypted. On a network that is not trusted, put a TLS proxy in front of it.

### 3.31 Find answers in documents (hybrid search)

People can add documents to a chat or to a knowledge base in Open WebUI. Open WebUI divides each document into passages. When a person asks a question, llmserver makes Open WebUI find the best passages in three steps:

1. Two searches find 20 candidate passages each: a keyword search (BM25) and a meaning search (the embedding model).
2. Open WebUI merges the two lists.
3. A reranker reads each candidate together with the question and gives it a score. The 6 best passages go to the model with the question.

The keyword search finds exact words, for example names, codes and numbers. The meaning search finds passages with different words. The reranker puts the passage that answers the question first.

Select the reranker:

| `LLMSERVER_WEBUI_RERANKER` | Reranker | Use it for |
|---|---|---|
| `fast` (default) | ms-marco-MiniLM-L6-v2 (91 MB), in Open WebUI on the CPU | English documents. About 1.5 s for 20 passages on two CPU cores |
| `best` | bge-reranker-v2-m3 (0.6 GB, Q8_0 GGUF), on llama.cpp through the gateway | Many languages and the best order. On a GPU, it is fast. On a CPU only, it is slow |
| `off` | None. The embedding model gives the order | Computers with very little memory |

1. Select the best reranker:

   ```bash
   llmserver config set webui_reranker best
   llmserver webui restart
   ```

   Result: llmserver starts `llama:gpustack/bge-reranker-v2-m3-GGUF:Q8_0` at a fixed commit, with the usual security checks, on a GPU if there is space. Open WebUI sends the passages to it through the gateway.

2. Make sure that it operates:

   ```bash
   llmserver ps
   ```

   Result: The reranker shows `rerank` in the `STATE` column. It is not in the menu, and Open WebUI does not show it as a chat model.

> **NOTE:** If llama.cpp is not enabled, or the reranker cannot start, llmserver uses the fast reranker and tells you why. `llmserver webui stop` also stops the reranker that it started.

> **CAUTION:** If you stop the best reranker (`llmserver stop all`, or a start that stops it to make space), questions about documents fail until you type `llmserver webui restart`. `llmserver webui status` tells you about this problem.

To use only the meaning search, set `llmserver config set webui_rag vector`. Then the reranker is not used.

> **NOTE:** Open WebUI keeps its settings in its database. The administrator can change them in **Admin Panel > Settings > Documents**. llmserver writes a setting into the database only when you change the related llmserver setting. Then the llmserver value replaces the change of the administrator.

> **NOTE:** After an upgrade from 2.6, the first start of Open WebUI writes all llmserver document and search settings once. Then make your changes in Open WebUI again, if you had some.

### 3.32 Search the web from the chat (SearXNG)

SearXNG is a search engine that asks public search engines (for example Google, Bing, DuckDuckGo and Brave) for you, without cookies. llmserver installs a private SearXNG for Open WebUI. Only Open WebUI can use it.

> **CAUTION:** The search questions go from this computer to public search engines. Open WebUI then reads the result pages from the internet. Turn on web search only if this is permitted for the people who use the chat.

1. Turn on web search:

   ```bash
   llmserver config set webui_search on
   llmserver webui restart
   ```

   Result: The first time, llmserver installs SearXNG (about 150 MB). Then it starts SearXNG on `127.0.0.1:8888`, in the sandbox, and starts Open WebUI.

2. In a chat, click **Web Search**. Then ask the question.

   Result: SearXNG finds 5 pages. Open WebUI reads them and puts them through the same hybrid search and reranker as documents. The answer shows the sources.

To turn off web search, type `llmserver config set webui_search off` and `llmserver webui restart`.

llmserver sets up SearXNG for safe use:
- It listens on `127.0.0.1` only. People on the network cannot use it directly.
- SearXNG has no releases. llmserver installs a fixed commit. It compares a SHA-256 of all program files with the value in the script, and it installs the Python packages from a lock file with a checksum for each file. If the archive host of GitHub is not available, llmserver gets the same commit with `git`, and does the same check.
- It gets a private secret key (`config/searxng-secret`, mode 600).
- llmserver writes its settings at each start (`webui/searxng/settings.yml`). To change them, for example the search engines, copy that file to `config/searxng.yml` and change the copy. llmserver then uses your copy.

> **NOTE:** If the port is in use, Open WebUI starts without web search and llmserver tells you. Set a different port with `llmserver config set webui_search_port N`.

### 3.33 Use one address for all models (the gateway)

Each model has its own port. The gateway gives all running models one OpenAI-compatible address, `http://127.0.0.1:8000/v1`. A client selects the model by its name, as `GET /v1/models` lists it.

1. Start the gateway (Open WebUI starts it automatically):

   ```bash
   llmserver gateway start
   ```

2. Give the address to the client, for example `OPENAI_BASE_URL=http://127.0.0.1:8000/v1`.

> **NOTE:** By default, the gateway accepts only clients on this computer. To make it available on the network, set `llmserver config set gateway_host 0.0.0.0`. Clients on the network then must send the API key (`llmserver env MODEL` shows it). Clients on this computer never need it.

> **NOTE:** The gateway log (`~/.llmserver/logs/gateway.log`) has one line for each request: the time, the client, the model and the result. It does not keep the text of the messages.

### 3.34 Upgrade from a version before 2.1

Versions before 2.1 kept their files in four places. At the first start, version 2.1 moves them into `~/.llmserver`:

| Old path | New path |
|---|---|
| `~/.local/share/llmserver/` | `~/.llmserver/` |
| `~/.llmserver_env` | `~/.llmserver/llmserver.env` |
| `~/.config/llmserver/` | `~/.llmserver/config/` |
| `~/.local/state/llmserver/install.log` | `~/.llmserver/logs/install.log` |
| `~/.local/state/llmserver/logs/` | `~/.llmserver/logs/models/` |
| `~/.local/state/llmserver/` (the rest) | `~/.llmserver/state/` |
| `~/.agents/skills/<name>` | `~/.llmserver/skills/<name>`, with links from `~/.agents/skills` and `~/.claude/skills` |

1. Stop all models:

   ```bash
   llmserver stop all
   ```

   > **NOTE:** While a model of the old version operates, llmserver does not move the files. It uses the old paths and shows a warning.

2. Run a command, for example `llmserver list`.

   Result: llmserver shows each move. The skills move when you next use `llmserver skills`, `llmserver doctor` or `llmserver upgrade`.

> **NOTE:** If `~/.llmserver_env` contains `LLMSERVER_HOME`, llmserver uses that folder for the move. After the move, set `LLMSERVER_HOME` in the environment or in `/etc/llmserver_env`.

---

## 4. Control the security of models and skills

### 4.1 How the security checks work

llmserver does a security check before it downloads or starts a model. The check has these steps:

1. It finds the exact commit of the model on Hugging Face.
2. It reads the size, the SHA-256 checksum and the scan results of each file.
3. It selects only the files that the engine needs.
4. It applies the rules (see [6.11](#611-security-rules)) and makes a decision: pass, ask or refuse.
5. It downloads the selected files at that commit.
6. It does a check of each file again: size, format and SHA-256.
7. It writes a check record.

Hugging Face scans files with ClamAV, a pickle scanner, Protect AI, JFrog and VirusTotal.

Each start uses the check record. The model starts from the recorded commit, without a connection to Hugging Face.

The engines get only the checked files:

| Engine | Files that llmserver downloads | Files that llmserver does not download |
|---|---|---|
| vLLM | Safetensors weights, configuration, tokenizer and chat template files | Pickle weights (`.bin`, `.pt`, `.pth`, `.pkl`), other formats (ONNX, TensorFlow, Flax), Python code, scripts, archives, programs |
| llama.cpp | The GGUF file of the selected quantization (all parts), and its `mmproj` file | All other files |
| Ollama | The model, through `ollama pull` | — |

> **NOTE:** vLLM starts with `--load-format safetensors` and in offline mode. It cannot load pickle weights or download other files.

> **NOTE:** llmserver refuses llama.cpp flags that download a model, for example `-hf` and `--model-url`. Give the model to llmserver instead.

> **NOTE:** For Ollama `hf.co/...` models, the pulled file must have the same SHA-256 as the checked GGUF file. For Ollama library models, no scan results exist. llmserver records the digests of the layers.

### 4.2 Select the security level

There are two levels:

| Item | `standard` (default) | `strict` |
|---|---|---|
| A file that Hugging Face flags as unsafe | Refused | Refused |
| A file that Hugging Face did not scan | Asks | Refused |
| A model that runs its own Python code | Asks for an approval | Refused without an approval |
| A start | Needs a check record | Needs an approved check record |
| `LLMSERVER_VERIFY=off` | Permitted | Not permitted |
| `LLMSERVER_ALLOW_PICKLE=on` | Permitted | Ignored |

1. Set the level:

   ```bash
   llmserver config set security strict
   ```

### 4.3 Do a check of a model before you download it

1. Do the check:

   ```bash
   llmserver trust check MODEL
   ```

   Result: llmserver shows the commit, the findings and the decision. It downloads nothing.

2. To get the full check as JSON, add `--json`.

### 4.4 Answer a security warning

A warning (`security warning: ... [ID]`) stops the start until you answer it.

1. In a terminal, read the warning.
2. Type `y` to continue, or push Enter to stop.

If there is no terminal, do one of these steps:

- To accept all warnings, add `--yes`.
- To accept one refused finding, add `--accept-risk ID`, for example `--accept-risk M-SCAN-PENDING`.

> **CAUTION:** An accepted risk stays in the check record. Do not accept a risk that you do not understand.

> **NOTE:** `--yes` does not approve the Python code of a model. Some findings cannot be accepted: an unsafe file, a dangerous pickle import, a wrong checksum and a wrong format.

### 4.5 Approve a model

Do this procedure for `SECURITY=strict`, or for a model that runs its own Python code.

1. Do a check of the model (see [4.3](#43-do-a-check-of-a-model-before-you-download-it)).
2. Approve the model:

   ```bash
   llmserver trust approve MODEL
   ```

   Result: llmserver downloads the files, does the check again and writes an approved record.

3. If the model runs its own Python code, approve the code for this commit:

   ```bash
   llmserver trust approve MODEL --remote-code
   ```

4. Read the code:

   ```bash
   llmserver trust show MODEL --code
   ```

> **CAUTION:** The Python code of a model operates with your user rights. Approve it only after you read it.

> **NOTE:** An approval is for one commit. A new commit needs a new approval.

### 4.6 Examine a check record

1. Show the record:

   ```bash
   llmserver trust show MODEL
   ```

   Result: The command shows the commit, the status, each file with its checksum and scan result, and the findings.

2. Show all records:

   ```bash
   llmserver trust list
   ```

3. Do a check of the checksums of all downloaded files:

   ```bash
   llmserver trust verify
   ```

   Result: The command shows each file that changed. The exit status is 3 if a file changed.

> **NOTE:** Each start compares the size of each file. `trust verify` and `doctor --deep` also compare the SHA-256 checksums.

### 4.7 Update a model to a new commit

1. Find the models that changed on Hugging Face:

   ```bash
   llmserver models outdated
   ```

2. Update a model:

   ```bash
   llmserver models update MODEL
   ```

   Result: llmserver does the check of the new commit, downloads it and writes a new record.

3. Start the model again to use the new commit:

   ```bash
   llmserver MODEL --restart
   ```

> **NOTE:** With `SECURITY=strict`, `models update` shows the command that approves the new commit.

> **NOTE:** llmserver never uses a new commit automatically. A model uses its recorded commit until you update it.

### 4.8 Lock a menu entry to a commit

1. Type `llmserver model edit`.
2. Add `@COMMIT` to the entry, for example `vllm:org/model@0123abcd...` or `llama:org/model-GGUF:Q4_K_M@0123abcd...`.
3. Save the file and close the editor.

To let llmserver write the commit after each check, type `llmserver config set lock_models on` (or `ask`).

### 4.9 Make the strict level necessary for all users

Do this procedure as the administrator.

1. Open `/etc/llmserver_env`.
2. Add this line:

   ```bash
   LLMSERVER_SECURITY_FLOOR=strict
   ```

3. Make sure that root owns the file and that only root can write it:

   ```bash
   sudo chown root: /etc/llmserver_env && sudo chmod 644 /etc/llmserver_env
   ```

   Result: Users cannot set a lower level.

> **NOTE:** llmserver ignores the floor if the file is not owned by root or if other users can write it. `llmserver doctor` shows a warning.

### 4.10 Use models that you downloaded before version 1.7

1. Start the model as usual.

   Result: llmserver does the check of the cached commit. It downloads only the files that are missing. Then it writes a record.

2. If Hugging Face is not available, answer the `M-HUB-OFFLINE` warning. llmserver records the checksums of the cached files.

> **NOTE:** The menu shows `cached, not checked yet` for these models.

### 4.11 Security checks for skills

llmserver does these checks before it installs a marketplace skill:

1. The skill must be in the Hugging Face skills marketplace.
2. The files must be text files of permitted types, with no links. Each file must be 1 MB or less, and the skill must be 5 MB or less.
3. llmserver reads the open issues of `github.com/huggingface/skills`. An issue about the skill with a security word refuses the skill.
4. A file that changed in the last 7 days needs an answer.
5. llmserver searches the files for commands that download and run code, for credential files, for hidden characters and for injected instructions.
6. llmserver shows the sites outside Hugging Face, GitHub and PyPI, the scripts, and the tools that the skill asks for.
7. llmserver records the SHA-256 checksum of each file.

The rules are in [6.11](#611-security-rules).

> **NOTE:** These checks find known patterns only. They do not prove that a skill is safe.

> **NOTE:** To use a copy of the marketplace (for example on a computer without internet), set `LLMSERVER_SKILLS_MIRROR` to a folder with the same layout: `marketplace.json` and `skills/NAME/`. The administrator of the copy is then responsible for its contents.

### 4.12 Move models to a computer without a network

Do this procedure on a computer with a network (A) and a computer without a network (B). Use the same llmserver version on A and B.

On computer A:

1. Download and check the models (see [4.5](#45-approve-a-model) for `SECURITY=strict`).
2. Write the bundle:

   ```bash
   llmserver export MODEL MODEL -o models.tar --sign ~/.ssh/id_ed25519
   ```

   Result: llmserver writes one tar file. It contains the checked files, their check records and a list of SHA-256 checksums.

3. Copy `models.tar` and the `llmserver` script to computer B.

On computer B:

4. If llmserver is not installed, install the engines from a bundle (see [4.13](#413-install-the-engines-without-a-network)).
5. Import the bundle:

   ```bash
   llmserver import models.tar
   ```

   Result: llmserver checks the signature and each file. Then it puts the models into the Hugging Face cache or the Ollama store.

6. Start a model as usual. It does not need a network.

> **CAUTION:** llmserver refuses a bundle if one file is different, missing or extra. Do not add files to a bundle.

> **NOTE:** With `SECURITY=strict`, only approved models can be exported, and computer B accepts only signed bundles (see [4.14](#414-sign-and-verify-bundles)).

> **NOTE:** The status of an imported model is `imported`. `llmserver trust show MODEL` shows the bundle and the status at export.

### 4.13 Install the engines without a network

On computer A (with a network, and the same type of CPU and GPU as computer B):

1. Install the engines that computer B needs.
2. Write a bundle with the engines:

   ```bash
   llmserver export -o engines.tar --with-engines
   ```

   Result: The bundle contains uv, Python, the engines and all Python packages, with their pinned checksums. You can add models to the same command.

On computer B:

3. Install from the bundle:

   ```bash
   llmserver install --from engines.tar
   ```

   Result: llmserver installs without a network. It does the same checksum and hash-lock checks as an installation from the network.

> **NOTE:** The bundle is for one CPU architecture and one llmserver version. llmserver refuses other combinations.

> **NOTE:** A vLLM bundle is large (10 GB or more), because it contains PyTorch and the CUDA or ROCm libraries.

### 4.14 Sign and verify bundles

On the computer that writes the bundles:

1. Make a key if you do not have one:

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/llmserver-bundles
   ```

2. Add `--sign ~/.ssh/llmserver-bundles` to each `llmserver export`.

On the computer that imports the bundles:

3. Make an `allowed_signers` file with one line for each permitted key:

   ```text
   ops@example namespaces="llmserver-bundle" ssh-ed25519 AAAA...
   ```

4. Set the file in `~/.llmserver/llmserver.env` (or in `/etc/llmserver_env`):

   ```bash
   LLMSERVER_TRUST_SIGNERS=/etc/llmserver/allowed_signers
   ```

   Result: `import` and `install --from` refuse bundles without a good signature from a key in the file.

> **NOTE:** The signature covers `SHA256SUMS`, and `SHA256SUMS` covers all other files. The check needs only `ssh-keygen` and `sha256sum`.

### 4.15 Make a software bill of materials

1. Write the bill of materials:

   ```bash
   llmserver versions --sbom > llmserver-sbom.json
   ```

   Result: The file is CycloneDX 1.6 JSON. It lists uv, Python, the engines, each locked Python package with its SHA-256 checksums, and each checked model.

2. Give the file to a vulnerability scanner, for example `trivy sbom llmserver-sbom.json` or `grype sbom:llmserver-sbom.json`.

### 4.16 Use local scanners

If these programs are on `PATH`, llmserver also uses them after each download:

| Program | Files | Result that refuses |
|---|---|---|
| `clamscan` (ClamAV) | All model files and all skill files | A virus signature (M-LOCAL-SCAN, S-LOCAL-SCAN) |
| `modelscan` | Pickle files (only with `LLMSERVER_ALLOW_PICKLE=on`) | Unsafe content |
| `gitleaks` | The code of a model, and the scripts of a skill | Secrets in the code |

1. Install the programs with the package manager of your system, if necessary.
2. Download a model as usual.

   Result: The check record shows `clean`, a finding, or `not-run` for each program.

> **NOTE:** llmserver does not install these programs. A missing program is recorded as `not-run`, never as `clean`.

> **NOTE:** To stop the local scanners, set `LLMSERVER_SCANNERS=off`.

### 4.17 Isolate the engines

If bubblewrap (`bwrap`) is installed, llmserver starts each engine in a sandbox:

- The system is read-only.
- The home folder is empty. The engine cannot read SSH keys, cloud credentials, tokens or your files.
- The model files are read-only. The Ollama store is writable.
- From the llmserver home, the engine sees only the engines and Python, read-only. It cannot see the settings, the API key, the logs or the records.
- Each engine has its own cache folder in `~/.llmserver/state/sandbox/`.
- `/tmp` is private.
- The GPUs and the network stay available. The server must listen on its port.

1. Install bubblewrap with the package manager of your system, for example:

   ```bash
   sudo apt install bubblewrap
   ```

2. Do a check:

   ```bash
   llmserver doctor
   ```

   Result: The line `sandbox` shows `engines run in a bubblewrap sandbox`.

3. Start a model as usual.

   Result: llmserver shows `sandbox: bubblewrap`. `llmserver ps --json` shows `"sandbox": "bwrap"`.

| `LLMSERVER_SANDBOX` | Result |
|---|---|
| `auto` (default) | Uses the sandbox if bubblewrap works. Otherwise it starts the engine without a sandbox and tells you |
| `on` | Refuses to start an engine without a sandbox |
| `off` | Never uses the sandbox |

> **NOTE:** Some systems do not permit user namespaces for normal users. Then bubblewrap does not work, and `doctor` tells you.

> **NOTE:** If an engine needs to write to a different folder, add the folder to `LLMSERVER_SANDBOX_RW` (absolute paths, separated by spaces).

> **NOTE:** The sandbox does not limit the network. llmserver starts the engines offline, but code in a model could still connect to the network.

### 4.18 Test the behaviour of a model

1. Start the model.
2. Do the built-in probes:

   ```bash
   llmserver eval MODEL
   ```

   Result: llmserver sends 9 requests. It shows how often the model followed an instruction hidden in a document, and how often it revealed a secret from its system prompt.

3. For more probes, use garak:

   ```bash
   llmserver eval MODEL --garak
   ```

   Result: The first time, llmserver asks to install garak (about 1.7 GB) into `~/.llmserver/tools`. Then garak runs a short set of prompt injection and jailbreak probes.

4. To select other garak probes, add `--probes`, for example `--probes promptinject,dan.DanInTheWild`.

The results go into the check record of the model (`llmserver trust show MODEL`).

> **CAUTION:** The probes find known attacks only. A model that passes can still behave badly. Do not use these results as a proof of safety.

> **NOTE:** garak can take a long time. Its output goes to `~/.llmserver/logs/garak.log`.

### 4.19 Limits of the security checks

The checks do not find all problems:

- A scan cannot find a model that was trained to behave badly (a backdoor).
- A scanner can miss a new attack.
- The Ollama library publishes no scan results.
- The checks do not control what a client or an agent does with the model or with a skill.
- A skill can be harmful in plain words that no pattern finds.
- A person with your user rights can change the files and the records.
- Without bubblewrap, an engine runs with all your user rights. With bubblewrap, it can still use the network.

---

## 5. Monitor and correct problems

### 5.1 Do a check of llmserver

1. Do a check:

   ```bash
   llmserver doctor
   ```

   Result: The command shows one line for each result. The levels are `ok`, `info`, `warn`, `problem` and `fixed`.

2. If the command shows a problem, repair it:

   ```bash
   llmserver doctor --fix
   ```

3. Do the check again.

   Result: The command shows `no problems found`.

`doctor` does a check of these items:

- the tools and the settings;
- the runtime and each engine (a change after installation, or incorrect permissions);
- the records of the models that operate;
- models that an interrupted start stopped;
- ports that two models use;
- temporary files, the lock, and the limits of the container;
- inference servers that llmserver did not start.

`doctor --fix` repairs the engines in two different ways:

- **Incorrect permissions:** `doctor --fix` removes the write permission for your group and for other users from all of the llmserver home. It does not download the engines again.
- **A changed engine program:** the checksum of the program is not equal to the checksum at installation. `doctor --fix` installs the engine again.

> **NOTE:** llmserver creates its files with umask `022`, whatever the umask of your shell is. If you give write permission to other users after installation, an engine does not start until you do `llmserver doctor --fix`.

> **NOTE:** `doctor` gives the exit status 1 when a problem remains. You can use it as a health check.

### 5.2 Recover after a crash of llmserver

If llmserver stops unexpectedly (for example `kill -9` or a power failure), do this procedure:

1. Do a check:

   ```bash
   llmserver doctor
   ```

2. Repair the problems:

   ```bash
   llmserver doctor --fix
   ```

   Result:
   - llmserver records again the servers that operate without a record (`ps` shows `adopted`).
   - It records a model as ready when the model answers.
   - It starts again the models that an interrupted start stopped. It does not do this if the new model operates.

> **NOTE:** If you push Ctrl-C during a start, llmserver cancels the start immediately. It starts again the models that it stopped.

### 5.3 Problems and corrective actions

| Message or symptom | Cause | Corrective action |
|---|---|---|
| `runs on Linux only` | The operating system is not Linux | Use Linux or WSL2 |
| `/proc and /sys must be mounted` | A container without them | Mount `/proc` and `/sys` |
| `the X engine is not installed` | The engine is not in the llmserver home | Type `llmserver install X` |
| `does not match its pinned checksum` | The file on the server is different | Nothing was installed. Try again later or from a different network |
| `publishes no SHA-256 digests` | `upgrade --latest` cannot verify the release | Keep the specified version |
| `writable by your group or other users` | The permissions of a file or folder in the llmserver home changed | Type `llmserver doctor --fix`. It removes the write permission and does not download again |
| `owned by` another user | A file in the llmserver home is not yours | Remove that file, or type `llmserver uninstall` and install again |
| `has changed since it was installed` | The checksum of an engine program changed | Type `llmserver doctor --fix` |
| `was not found on Hugging Face` (exit 3, `M-NOT-FOUND`) | The name is not correct, or the model is private | Correct the name. llmserver shows similar names if they exist. For a private model, type `llmserver hf auth login` |
| `failed the security check above` (exit 3) | A rule refused the model. The line `security check failed: ... [ID]` gives the reason | Read the reason. Use a different model, or add `--accept-risk ID` if llmserver shows this option |
| `security warning` ... `no terminal to ask` (exit 4) | A warning needs an answer | Add `--yes`, or start the model in a terminal |
| `the best reranker did not start ... using the fast one` | `webui_reranker best` could not start bge-reranker-v2-m3 (no space, the strict level, or llama.cpp not enabled) | Read the lines above the message. With the strict level, type `llmserver trust approve llama:gpustack/bge-reranker-v2-m3-GGUF:Q8_0@3093af03b1a635e67b084b1d8c03c5f5e020fd05`. Then type `llmserver webui restart` |
| Questions about documents fail in Open WebUI | The best reranker stopped | Type `llmserver webui restart` |
| `web search is off` (when Open WebUI starts) | SearXNG did not start, or its port is in use | Read `logs/searxng.log`, or set a different port: `llmserver config set webui_search_port N` |
| `the SearXNG source does not match llmserver's pin` | The downloaded source is different from the fixed commit | Nothing was installed. Try again later, or check `LLMSERVER_GITHUB` |
| Web search in Open WebUI finds nothing | The search engines refuse the questions, or this computer has no internet access | Read `logs/searxng.log`. Change the search engines in `config/searxng.yml` (see [3.32](#332-search-the-web-from-the-chat-searxng)) |
| `is not a model name` | The name has an incorrect form, or the engine name is incorrect | Use `[engine:]owner/model`. The engines are `llama`, `vllm` and `ollama` |
| `this is a GGUF repository ... Serve it with llama.cpp instead` (`M-NO-WEIGHTS`) | vLLM cannot serve GGUF files | Use the command that llmserver shows, for example `llmserver llama:ORG/REPO:QUANT`. To correct the menu, type `llmserver model rm` and `llmserver model add` |
| `N%-on-cpu` in `llmserver ps` | Ollama did not have space on the GPUs for all of the model | Stop other models (`llmserver stop MODEL`), then `llmserver MODEL --restart` |
| `the benchmark request failed (...)` | The server did not answer the measurement | Read the reason in the brackets. `llmserver chat MODEL "hi"` shows if the model answers at all |
| `is not a command (did you mean ...)` | A command name is not correct | Use the command that llmserver shows. To start an Ollama model with that name, write `ollama:NAME` |
| `skipping line N of .../config/models` | A menu line is not correct | Type `llmserver model edit` and correct the line, or `llmserver model rm` |
| `LLMSERVER_X must be ...` (every command stops) | A setting has an incorrect value | Type `llmserver config set X VALUE` with a correct value, or `llmserver config unset X` |
| `is not a setting (did you mean ...)` | The setting name is not correct | Use the name that llmserver shows. Write engine variables in capitals |
| `memory check skipped` | The model is not downloaded, and llmserver could not get its size | No action. To do the check, download the model first: `llmserver download MODEL` |
| `runs its own Python code` (exit 4) | The model needs `--trust-remote-code` | Read the code. Then type `llmserver trust approve MODEL --remote-code` |
| `SECURITY=strict: ... no approved check record` (exit 4) | The strict level needs an approval | Type `llmserver trust approve MODEL` |
| `M-HUB-OFFLINE` | Hugging Face is not available | Connect to the network. A model with a check record starts without the network |
| `M-ACCESS` | The model is gated, and you did not accept its terms or log in | Accept the terms on Hugging Face. Then type `llmserver hf auth login`, or set `HF_TOKEN` in `~/.llmserver/llmserver.env` |
| `M-HASH`, `M-FORMAT` or `M-DIGEST` | A downloaded file is not the checked file | No action. llmserver deleted the file. Try again later or from a different network |
| `would download a model that llmserver did not check` | A llama.cpp flag such as `-hf` | Remove the flag. Give the model as the llmserver model or `--draft` |
| `changed size since it was checked` | A model file changed after the check | No action. llmserver does the check and downloads the file again |
| `differ from the check record` in `doctor --deep` | A file changed, but not its size | Type `llmserver doctor --deep --fix`. Then start the model |
| `is not in Hugging Face's skills marketplace` (S-NOT-LISTED) | The skill name is not listed | Type `llmserver skills list` and use a listed name |
| `S-GITHUB-OFFLINE` | llmserver cannot read the GitHub issues | Connect to the network, or add `--yes` after you read the skill. Set `GITHUB_TOKEN` if GitHub limits the requests |
| `exists and was not installed by llmserver` | A skill with the same name is there | Keep it, or add `--force` to replace it |
| `was changed after llmserver installed it` | You changed the skill | Keep your changes, or add `--force` |
| `the hf CLI is pinned by llmserver` | `llmserver hf update` | Type `llmserver upgrade runtime` |
| `is not signed, and LLMSERVER_TRUST_SIGNERS requires a signature` | The bundle has no signature | Export it again with `--sign` |
| `is signed by a key that is not in` | The key is not permitted | Add the key to the `allowed_signers` file, or use a different bundle |
| `do not match SHA256SUMS` or `does not list` | The bundle changed after export | Get the bundle again. Do not use the changed bundle |
| `the bundle does not contain` | The bundle is for a different machine or GPU | Export again on a machine of the same type |
| `bubblewrap does not work here` | `LLMSERVER_SANDBOX=on`, and the sandbox is not available | Install bubblewrap, or permit user namespaces, or set `LLMSERVER_SANDBOX=auto` |
| An engine in the sandbox cannot write a file | The folder is read-only in the sandbox | Add the folder to `LLMSERVER_SANDBOX_RW`, or set `LLMSERVER_SANDBOX=off` for a test |
| `garak is not installed` | `eval --garak` without a terminal | Add `--yes`, or type the command in a terminal |
| `M-LOCAL-SCAN` or `S-LOCAL-SCAN` | A local scanner found a problem | Read the message of the scanner. Use a different model or skill |
| `the check record was revoked` | Somebody withdrew the approval | Type `llmserver trust approve MODEL` if the model is safe |
| `PyTorch was resolved for 'cpu' instead of 'cuda'` | uv did not find the NVIDIA driver | Set `UV_TORCH_BACKEND=cu130`. Then type `llmserver install vllm` |
| `the Vulkan build sees no GPU` | The Vulkan driver is missing | Install the Vulkan part of the graphics driver |
| `needs glibc 2.39` | The C library is too old | Set `LLMSERVER_LLAMA_BUILD=vulkan`, or use your own engine (see [3.18](#318-use-an-engine-that-you-installed)) |
| `needs these models stopped ... no terminal` | The GPUs are full and llmserver cannot ask | Use `--evict` or `--offload`, or stop a model |
| `LLMSERVER_EVICT=never` | Eviction is off | Stop a model, or use `--evict` |
| `too many of them are pinned` | All models that operate are pinned | Remove a pin, or increase `LLMSERVER_MAX_MODELS` |
| `will not run well enough on this machine` | The model is too large | Type `llmserver check MODEL` and use a recommendation |
| Verdict `UNKNOWN` | No registry access, or no runtime | Connect to the network, or type `llmserver install runtime` |
| `is already running` | The model operates | Use it, or use `--restart` |
| `no free port` | All ports in the range are in use | Stop a model, or increase `LLMSERVER_PORT_SPAN` |
| `several models are running; say which` | The command needs a model | Give a menu number, a name or a port |
| `refusing to serve ... without authentication` | Network access without a key | Set `LLMSERVER_AUTH=auto` |
| `accepted a request without the API key` | The engine ignored the key | Use `--api-key-file` after `--` |
| `was not healthy after 900s` | The model loads slowly | Increase `LLMSERVER_READY_TIMEOUT` |
| `checksum MISMATCH` two times | The download is damaged | Do a check of the disk and the network |
| `found X running ... without a record` | An earlier llmserver stopped unexpectedly | No action. llmserver records the server again |
| `an earlier start was interrupted` | llmserver stopped after it stopped models | Type `llmserver doctor --fix` |
| `not in the inherited CUDA_VISIBLE_DEVICES` | A GPU mask excludes the GPU | No action, or change the mask |
| `renderD... is not accessible` | The container cannot open the AMD GPU | Pass `/dev/dri` to the container |
| The RAM in `check` is less than the computer has | A cgroup memory limit | No action. llmserver obeys the limit |
| Clients get 401 | The client sends no key | Type `llmserver env MODEL` |
| A model shows `starting` for a long time | The model loads | Type `llmserver logs MODEL` |

---

## 6. Reference data

### 6.1 Commands

| Command | Function |
|---|---|
| `llmserver [MODEL]` | Starts a model. Without a model, it shows the menu |
| `ps` | Shows the models that operate |
| `capacity` | Shows the GPU memory, the models and the space for more models |
| `check [MODEL]` | Shows the hardware and the verdicts |
| `list` | Shows the menu, with the cache state and the verdict of each entry |
| `model list` | Shows the numbered menu (fast, no network) |
| `model add MODEL [-- FLAGS]` | Makes sure that the model exists, then adds it to the menu |
| `model rm [N\|MODEL...]` | Removes menu entries. Without an entry, it asks which. `--purge` also deletes the downloaded files |
| `model purge MODEL` | Deletes the files that llmserver downloaded for a model, and its check records |
| `model edit` | Opens the menu in your editor, then does a check of it |
| `config [list]`, `config get NAME` | Shows the settings and the source of each value |
| `config set NAME VALUE`, `config unset NAME` | Changes or removes one setting, after a check |
| `config edit [tuning]` | Opens the settings (or the tuning rules) in your editor, then does a check |
| `key show`, `key new` | Shows where the API key is, or makes a new key |
| `autostart [list]`, `autostart on\|off MODEL...` | Starts models when you log in (systemd user services) |
| `stop MODEL` or `stop all` | Stops one model or all models |
| `pin MODEL`, `unpin MODEL` | Protects a model from a stop, or removes the protection |
| `webui [start\|stop\|restart\|status]` | Open WebUI, a web chat for people on the network. Installs it on first use. `restart` applies changed settings |
| `gateway [start\|stop\|status]` | One OpenAI-compatible address for every running model |
| `chat [MODEL] [MESSAGE]` | Starts a chat with a model in the terminal. With a message, or a pipe, it answers once. `--system TEXT` sets a system prompt |
| `env [MODEL]` | Shows the client variables |
| `logs [MODEL]` | Shows the output of a model |
| `log` | Shows the activity log |
| `bench [MODEL]` | Measures the speed of a model |
| `download [MODEL]` | Does the security checks and downloads a model |
| `trust check MODEL` | Does the security checks. Downloads nothing |
| `trust approve MODEL` | Does the checks, downloads the model and approves it. `--remote-code` also approves its Python code |
| `trust revoke MODEL` | Withdraws the approval and stops the model |
| `trust show MODEL` | Shows the check record. `--code` shows the Python code of the model |
| `trust verify [MODEL]` | Compares the SHA-256 checksums of the downloaded files with the records |
| `trust list` | Shows all check records |
| `models outdated` | Shows the models that changed on Hugging Face |
| `models update MODEL` | Does the checks of the new commit and downloads it |
| `skills [list]` | Shows the installed, the generated and the marketplace skills |
| `skills check NAME`, `skills preview NAME` | Does the checks of a skill, or shows it. Installs nothing |
| `skills add [NAME...]` | Does the checks and installs skills. Without a name: `llmserver` and `hf-cli` |
| `skills update [NAME...]` | Updates the skills that llmserver installed |
| `skills remove NAME...` | Removes skills that llmserver installed |
| `export MODEL... -o FILE` | Writes checked models into a bundle. `--with-engines` adds the engines. `--sign KEY` signs it |
| `import FILE` | Checks a bundle and puts its models in place |
| `install --from FILE` | Installs the engines from a bundle, without a network |
| `versions --sbom` | Writes a CycloneDX bill of materials |
| `eval [MODEL]` | Probes a running model for prompt injection and system prompt leaks. `--garak` also runs garak |
| `hf ARGS...` | Starts the fixed Hugging Face CLI. `--link` and `--unlink` add or remove `~/.local/bin/hf` |
| `setup` | Does the setup |
| `install [COMPONENT...]` | Installs `runtime`, `llama`, `ollama`, `vllm`, `all` (the engines), `webui` or `searxng` |
| `upgrade [COMPONENT]` | Installs the specified versions, or with `--latest` the newest verified versions |
| `rollback COMPONENT` | Goes back to the previous build |
| `versions` | Shows what is installed |
| `doctor` | Does a check of llmserver. `--fix` repairs. `--deep` also compares the checksums of the model files |
| `uninstall` | Removes llmserver |

### 6.2 Options

| Option | Function |
|---|---|
| `-e`, `--engine NAME` | Selects the engine for a model name without a prefix |
| `--host ADDR` | Sets the address for this start |
| `--port N` | Sets the port |
| `-c`, `--ctx N` | Sets the context: a number, `fit` or `auto` |
| `--restart` | Starts a model that operates again, on the same port |
| `--evict` | Stops old models without a question if necessary |
| `--offload` | Starts the model partially on the CPU instead of a stop of other models |
| `--foreground` | Shows the output. Ctrl-C stops the model |
| `-n`, `--dry-run` | Shows the plan. Changes nothing |
| `--insecure` | Permits network access without authentication for this start |
| `--no-vram-guard` | Does not refuse a model for hardware reasons |
| `--no-preflight` | Does not check the llama.cpp flags |
| `--draft MODEL`, `--draft-n N`, `--no-draft` | Controls speculative decoding |
| `--json` | Writes JSON (`ps`, `check`, `capacity`, `list`, `doctor`, `versions`, `trust`, `models`) |
| `--accept-risk ID` | Accepts one refused finding. The record keeps it. You can use it more than one time |
| `--remote-code` | Approves the Python code of a model (`trust approve`) |
| `--code` | Shows the Python code of a model (`trust show`) |
| `--deep` | Compares the checksums of all model files (`doctor`) |
| `--project` | Uses the current folder instead of your home folder (`skills`) |
| `-o`, `--output FILE` | The bundle file (`export`) |
| `--with-engines` | Adds the engines to the bundle (`export`) |
| `--sign KEY` | Signs the bundle with an ssh private key (`export`) |
| `--from FILE` | Installs from a bundle (`install`) |
| `--sbom` | Writes the bill of materials (`versions`) |
| `--garak`, `--probes LIST` | Also runs garak, with these probes (`eval`) |
| `--force` | Replaces or removes a changed skill, or a skill that llmserver did not install (`skills`) |
| `--fix` | Repairs (`doctor`) |
| `--latest` | Uses the newest verified versions (`upgrade`) |
| `-y`, `--yes` | Accepts without a question (`setup`, `install`, `uninstall`, security warnings). It does not approve the Python code of a model |
| `-v`, `--verbose` | Shows more details |
| `--runs N`, `--long N`, `--gen N`, `--history` | Controls `bench` |
| `-f`, `--errors`, `--lines N` | Controls `log` |
| `--purge-models`, `--keep-config`, `--keep-script` | Controls `uninstall` |
| `--purge` | With `model rm`: also deletes the files that llmserver downloaded for the entries |
| `--system TEXT` | With `chat`: the system prompt |
| `--think` | With `chat`: shows the thinking of a reasoning model |
| `--reasoning LEVEL` | With `chat` and `bench`: `low`, `medium`, `high` or `none`; sent as `reasoning_effort` |
| `--` | Sends the next words to the engine |

### 6.3 Model names

| Name | Engine | Model |
|---|---|---|
| `llama:ORG/REPO[:QUANT][@COMMIT]` | llama.cpp | GGUF repository on Hugging Face |
| `vllm:ORG/REPO[@REVISION]` | vLLM | Hugging Face repository |
| `ollama:NAME[:TAG]` | Ollama | Ollama library model |
| `ollama:hf.co/ORG/REPO[:QUANT]` | Ollama | GGUF repository on Hugging Face |

If you do not give an engine, llmserver selects it:
- Names with `gguf` or a `:QUANT` go to llama.cpp.
- Names without `/` go to Ollama.
- Other Hugging Face names go to vLLM.

### 6.4 Settings

Put the settings in `~/.llmserver/llmserver.env`.

| Setting | Default | Function |
|---|---|---|
| `LLMSERVER_ENGINES` | installed engines | The engines that llmserver uses |
| `LLMSERVER_DEFAULT_ENGINE` | empty | The engine for names that more than one engine can operate |
| `LLMSERVER_HOST` | `127.0.0.1` | The network address |
| `LLMSERVER_PORT` | `8080` | The first port |
| `LLMSERVER_PORT_SPAN` | `100` | The number of ports after the first port |
| `LLMSERVER_AUTH` | `auto` | `auto`, `key` or `none` |
| `LLMSERVER_ALLOW_INSECURE` | `0` | `1` permits network access without authentication |
| `LLMSERVER_CTX` | `32768` | The context of each model: a number, `fit` or `auto` |
| `LLMSERVER_MAX_MODELS` | `0` | The maximum number of models. `0` is no limit |
| `LLMSERVER_EVICT` | `ask` | `ask`, `lru` or `never` |
| `LLMSERVER_GPUS` | empty | The GPUs that llmserver can use |
| `LLMSERVER_LLAMA_FLAGS` | `--jinja` | Flags for each llama.cpp start |
| `LLMSERVER_VLLM_FLAGS` | empty | Flags for each vLLM start |
| `LLMSERVER_OLLAMA_ATTACH` | empty | `host:port` of an Ollama server to use |
| `LLMSERVER_OLLAMA_KEEP_ALIVE` | `-1` | The time that Ollama keeps a model. `-1` is always |
| `LLMSERVER_DRAFT_N` | `4` | The tokens that a draft model proposes |
| `LLMSERVER_NUMA` | `auto` | `auto` or `off` |
| `LLMSERVER_TUNING` | `on` | `on` or `off`: the rules in `~/.llmserver/config/tuning` |
| `LLMSERVER_VRAM_GUARD` | `enforce` | `off`, `warn`, `enforce` or `strict` (see [6.7](#67-vram-guard)) |
| `LLMSERVER_VRAM_RESERVE_MB` | `512` | The free GPU memory to keep on each GPU |
| `LLMSERVER_VERIFY` | `require` | `require` or `off`: the SHA-256 check of each model file. `strict` needs `require` |
| `LLMSERVER_SECURITY` | `standard` | `standard` or `strict` (see [4.2](#42-select-the-security-level)) |
| `LLMSERVER_SECURITY_FLOOR` | empty | Only in `/etc/llmserver_env`, owned by root: the lowest level that users can set |
| `LLMSERVER_ALLOW_PICKLE` | `off` | `on` permits vLLM models that have only pickle weights. Not with `strict` |
| `LLMSERVER_SANDBOX` | `auto` | `auto`, `on` or `off`: run the engines in a bubblewrap sandbox (see [4.17](#417-isolate-the-engines)) |
| `LLMSERVER_GATEWAY_HOST`, `LLMSERVER_GATEWAY_PORT` | `127.0.0.1`, `8000` | The address of the gateway (see [3.33](#333-use-one-address-for-all-models-the-gateway)) |
| `LLMSERVER_WEBUI`, `_WEBUI_HOST`, `_WEBUI_PORT`, `_WEBUI_AUTH` | `ask`, `0.0.0.0`, `3000`, `on` | Open WebUI (see [3.30](#330-give-people-on-the-network-a-web-chat-open-webui)) |
| `LLMSERVER_WEBUI_RAG` | `hybrid` | `hybrid` (keywords and meaning, then the reranker) or `vector` (see [3.31](#331-find-answers-in-documents-hybrid-search)) |
| `LLMSERVER_WEBUI_RERANKER` | `fast` | `fast` (CPU), `best` (bge-reranker-v2-m3 on llama.cpp) or `off` |
| `LLMSERVER_WEBUI_SEARCH`, `_WEBUI_SEARCH_PORT` | `off`, `8888` | Web search through a private SearXNG on `127.0.0.1` (see [3.32](#332-search-the-web-from-the-chat-searxng)) |
| `LLMSERVER_SANDBOX_RW` | empty | More folders that the engines can write in the sandbox (absolute paths) |
| `LLMSERVER_SCANNERS` | `auto` | `auto` uses `clamscan`, `modelscan` and `gitleaks` if they are on `PATH`. `off` |
| `LLMSERVER_TRUST_SIGNERS` | empty | An ssh `allowed_signers` file. Bundles must have a good signature from one of its keys. Necessary with `strict` |
| `LLMSERVER_SKILLS` | `ask` | `ask`, `on` or `off`: `setup` offers, installs or skips the generated skills |
| `LLMSERVER_SKILLS_MIRROR` | empty | A folder with a copy of the Hugging Face skills marketplace |
| `LLMSERVER_LOCK_MODELS` | `off` | `off`, `ask` or `on`: writes the checked commit into the menu entry |
| `LLMSERVER_READY_TIMEOUT` | `900` | The seconds to wait for a model |
| `LLMSERVER_STOP_TIMEOUT` | `20` | The seconds before a forced stop |
| `LLMSERVER_LOG_LEVEL` | `info` | `info` or `debug` |
| `LLMSERVER_LOG_MAX_MB` | `5` | The maximum size of `install.log` |
| `LLMSERVER_LLAMA_SOURCE`, `_VLLM_SOURCE`, `_OLLAMA_SOURCE` | `managed` | `managed`, `system` (from `PATH`), or the full path of your own program or its folder |
| `LLMSERVER_LLAMA_BUILD` | `auto` | `auto`, `cpu`, `vulkan`, `cuda-12.8` or `cuda-13.4` |
| `LLMSERVER_AUTO_INSTALL` | `on` | `on` installs the runtime when necessary |
| `LLMSERVER_HOME` | `~/.llmserver` | The llmserver home. Only in the environment or in `/etc/llmserver_env` |
| `LLMSERVER_MODEL_STORE` | `shared` | `shared`: the model files go into the shared caches. `private`: into `models/` in the llmserver home. `HF_HOME`, `HF_HUB_CACHE` and `OLLAMA_MODELS` have priority |
| `LLMSERVER_GITHUB`, `LLMSERVER_GITHUB_API`, `LLMSERVER_PYPI`, `LLMSERVER_VLLM_ROCM_INDEX`, `LLMSERVER_FLASHINFER_INDEX` | the public servers | The download and update servers |
| `LLMSERVER_SYSFS`, `LLMSERVER_PROCFS`, `LLMSERVER_DEVFS` | `/sys`, `/proc`, `/dev` | The locations of the system information |

You can also put engine variables in the file, for example `HF_TOKEN`, `HF_HOME`, `OLLAMA_MODELS` and `UV_INDEX_URL`.

### 6.5 Files and folders

All paths in the first table are in the llmserver home, `~/.llmserver` (mode 700).

| Path | Contents |
|---|---|
| `llmserver.env` | The settings (mode 600) |
| `config/models` | The menu |
| `config/api-key` | The API key (mode 600) |
| `config/tuning` | The tuning rules |
| `logs/install.log` | The activity log |
| `logs/models/` | The output of each model |
| `logs/ollama-pull.log`, `logs/garak.log` | The output of Ollama downloads and of garak |
| `logs/gateway.log`, `logs/webui.log`, `logs/searxng.log` | The requests through the gateway (no message text), and the output of Open WebUI and SearXNG |
| `state/services/` | One record each for the gateway, Open WebUI and SearXNG when they operate |
| `webui/` | Open WebUI: `data/` (accounts, chats, uploads), its embedding model and reranker, its home folder, the generated SearXNG settings, and `llmserver-applied.json` (the values llmserver last wrote into its settings) |
| `config/webui-secret`, `config/searxng-secret` | The secret keys of Open WebUI and SearXNG (mode 600) |
| `config/searxng.yml` | Your SearXNG settings, if you made this file. Else llmserver writes its own |
| `tools/open-webui/`, `tools/searxng/` | Open WebUI and SearXNG with their Python packages, only after you install them |
| `state/running/` | One record for each model that operates |
| `state/ports`, `state/pins` | The port of each model, and the pinned models |
| `state/rollback.*` | Records of models that a start stopped |
| `state/sandbox/` | The private cache folder of each engine in the sandbox |
| `skills/` | The skills for AI agents |
| `trust/records/model/` | One check record (JSON) for each model and commit |
| `trust/records/skill/` | One record (JSON) for each installed skill, with the checksum of each file and its links |
| `engines/`, `runtime/`, `python/`, `uv/`, `bin/` | The engines, the helper runtime, Python and uv |
| `tools/garak/` | garak and its results, only after `llmserver eval --garak` |
| `pins.local` | Newer pins from `llmserver upgrade --latest` |
| `cache/`, `tmp/`, `locks/` | Downloads, work files and locks of llmserver |
| `models/` | The model files, only with `LLMSERVER_MODEL_STORE=private` |
| `.llmserver-home` | The marker file. llmserver deletes or changes only a folder with this file |

Other paths:

| Path | Contents |
|---|---|
| `/etc/llmserver_env` | Optional settings for all users |
| `~/.agents/skills/`, `~/.claude/skills/` | Links to the skills in the home (`CLAUDE_CONFIG_DIR` changes the second folder) |
| `~/.local/bin/hf` | The `hf` link, only after `llmserver hf --link` |
| `~/.cache/huggingface/hub/` | The Hugging Face models (other programs share them) |
| `~/.ollama/models/` | The Ollama models (other programs share them) |

### 6.6 Verdicts

| Verdict | Meaning |
|---|---|
| GOOD | The model fits on the GPU with 10% free memory. It operates well |
| TIGHT | The model fits on the GPU with less free memory. Long prompts can fail |
| OFFLOAD | Part of the model operates on the CPU. It operates slowly |
| CPU | There is no applicable GPU. The model operates slowly on the CPU |
| NO | The model cannot operate on this computer |
| UNKNOWN | llmserver cannot find the size of the model |

### 6.7 VRAM guard

| `LLMSERVER_VRAM_GUARD` | NO | TIGHT, OFFLOAD | CPU |
|---|---|---|---|
| `off` | Starts | Starts | Starts |
| `warn` | Starts with a warning | Starts with a warning | Starts |
| `enforce` | Refuses | Starts with a warning | Starts |
| `strict` | Refuses | Refuses | Starts |

### 6.8 Engine builds

| Engine | Version | Build that llmserver selects |
|---|---|---|
| llama.cpp | b11798 | CUDA 13.4 with NVIDIA driver 580 or newer and compute capability 7.5 or newer. CUDA 12.8 with driver 570 or newer (x86-64). Vulkan for AMD, for mixed GPUs or for older drivers. CPU without a GPU |
| Ollama | v0.40.7 | The Linux build, plus the ROCm libraries for AMD GPUs |
| vLLM | 0.32.0 | CUDA for NVIDIA. ROCm 7.2.3 for AMD on x86-64. vLLM is not available without a GPU |
| Runtime | uv 0.13.4, Python 3.14.8 | One build for all computers |

### 6.9 Exit status

| Status | Meaning |
|---|---|
| 0 | Success |
| 1 | Error. The message tells you what changed. For `ps`, no model operates. For `doctor`, a problem remains |
| 3 | The security checks refused the model or the skill, `trust verify` found a changed file, or a bundle failed its checks |
| 4 | The security checks need an answer or an approval, and there was no terminal or approval |

### 6.10 Limits

- Each model has its own port. llmserver does not route requests between models.
- llmserver divides GPU memory between models. It does not divide GPU computation.
- On AMD, llmserver measures the memory of a model one time, at the start.
- llama.cpp has no Linux ROCm build. On AMD, llmserver uses the Vulkan build.
- llmserver gets the vLLM lock file from the package servers at the first installation.
- On NVIDIA, llmserver also installs the precompiled FlashInfer kernels for vLLM, with a second lock file (`vllm-<version>-cuda-kernels.txt`). It does not use `vllm download-kernels`, because that command does not do a check of the downloads.
- You cannot move the llmserver home without a new installation.
- The security checks have limits (see [4.19](#419-limits-of-the-security-checks)).

### 6.11 Security rules

"Accept" tells you how to continue after a finding: `--accept-risk ID`, an approval, or not possible.

| ID | Condition | `standard` | `strict` | Accept |
|---|---|---|---|---|
| M-SCAN-UNSAFE | Hugging Face flags a selected file as unsafe | Refuse | Refuse | Not possible |
| M-PICKLE-IMPORT | A pickle file imports a dangerous object | Refuse | Refuse | Not possible |
| M-SCAN-WARN | A scan gives a warning | Ask | Refuse | `--accept-risk` |
| M-SCAN-PENDING | A selected file has no scan result yet | Ask | Refuse | `--accept-risk` |
| M-PICKLE-ONLY | A vLLM model has only pickle weights | Refuse | Refuse | `LLMSERVER_ALLOW_PICKLE=on` or `--accept-risk` (standard only) |
| M-REMOTE-CODE | The model runs its own Python code | Ask for an approval | Refuse | `trust approve --remote-code` |
| M-REMOTE-XREPO | The code comes from a different repository | Refuse | Refuse | Not possible |
| M-SIZE | The weights are larger than the parameter count explains (more than 16 bits for each parameter, or more than the safetensors data types) | Ask | Refuse | `--accept-risk` |
| M-EXTRA-EXEC | The repository has code, scripts, archives or programs. llmserver does not download them | Note | Ask | `--accept-risk` |
| M-NO-LICENSE | The model card has no license | Note | Ask | `--accept-risk` |
| M-REGISTRY | An Ollama library model has no scan results | Note | Ask | `trust approve` |
| M-HUB-OFFLINE | Hugging Face is not available | Ask | Refuse | `--accept-risk` |
| M-ACCESS | The model is gated, and access is not given | Refuse | Refuse | Log in |
| M-NOT-FOUND | The model, the commit or the quantization does not exist, or the model is private and you did not log in. The message shows similar names | Refuse | Refuse | Not possible |
| M-NO-WEIGHTS | The repository has no weights that vLLM can load | Refuse | Refuse | Not possible |
| M-HASH | A downloaded file has a wrong size or checksum | Refuse | Refuse | Not possible |
| M-FORMAT | A downloaded file has a wrong format, hidden data or is a program | Refuse | Refuse | Not possible |
| M-DIGEST | Ollama pulled a different file than the checked file | Refuse | Refuse | Not possible |
| M-NO-HASH | Hugging Face gave no checksum. llmserver records the local checksum | Note | Refuse | Not possible |
| M-LOCAL-FILE | The model is a file on this computer, not from Hugging Face: there are no scan results. llmserver checks the file itself | Ask | Ask, then approval | `--accept-risk` |
| M-ATTACHED | An attached Ollama server keeps its files where llmserver cannot read them | Note | Refuse | `--accept-risk` |
| M-SKIPPED, M-GATED | Information | Note | Note | — |
| S-NOT-LISTED | The skill is not in the Hugging Face skills marketplace | Refuse | Refuse | Not possible |
| S-FILETYPE, S-LINK, S-SIZE, S-NO-SKILLMD | A binary file, a file type that skills do not need, a link, a path outside the skill, a file over 1 MB, a skill over 5 MB, or no `SKILL.md` | Refuse | Refuse | Not possible |
| S-HUB-OFFLINE | The marketplace is not available | Refuse | Refuse | Not possible |
| S-ISSUE-SECURITY | An open GitHub issue reports a security problem with the skill | Refuse | Refuse | `--accept-risk` |
| S-ISSUE-OPEN | An other open GitHub issue names the skill | Note | Ask | `--accept-risk` |
| S-GITHUB-OFFLINE | The GitHub issues cannot be read | Ask | Refuse | `--accept-risk` |
| S-RECENT | A file changed in the last 7 days | Ask | Refuse | `--accept-risk` |
| S-PATTERN | A command downloads and runs code, decodes and runs code, or reads credential files | Ask | Ask | `--accept-risk` |
| S-UNICODE | Hidden characters, or an instruction to ignore earlier instructions | Ask | Refuse | `--accept-risk` |
| S-DOMAINS | Sites outside Hugging Face, GitHub and PyPI | Note | Ask | `--accept-risk` |
| S-TOOLS | The skill asks for tools that the agent can use without a question | Note | Ask | `--accept-risk` |
| S-SCRIPTS | The skill has scripts (llmserver does not run them) | Note | Note | — |
| M-LOCAL-SCAN, S-LOCAL-SCAN | A local scanner (`clamscan`, `modelscan`, `gitleaks`) found a problem | Refuse | Refuse | `--accept-risk` |

### 6.12 Check record

Each check record is a JSON file in `~/.llmserver/trust/records/model/`. It contains:

- the engine, the model, the reference (for example `main`) and the commit;
- the status: `accepted`, `approved` or `revoked`, with the user and the time;
- each selected file: path, size, SHA-256 or git checksum, and the Hugging Face scan results;
- the files that llmserver did not download, with their type;
- the findings and the accepted risks;
- the remote code approval;
- the security level and the llmserver version.

A skill record also contains the scope, the folder, the link and the checksum of each file. For a generated skill, it contains the versions that generated it.

### 6.13 Bundle format

A bundle is an uncompressed tar file with only plain files:

| Member | Contents |
|---|---|
| `SHA256SUMS` | The SHA-256 checksum of each other member (the `sha256sum` format) |
| `SHA256SUMS.sig` | Optional: the ssh signature of `SHA256SUMS`, namespace `llmserver-bundle` |
| `manifest.json` | The bundle ID, the creator, the time, the llmserver version, the architecture, and where each model file goes |
| `records/model/*.json` | The check records of the models |
| `blobs/sha256/<checksum>` | The model files, named by their SHA-256 checksum |
| `engines/` | With `--with-engines`: `files/` (uv and engine archives), `python/`, `wheels/`, `locks/`, `engines.json` |

---

## 7. Undo, restore and remove

### 7.1 Stop models

1. Stop one model:

   ```bash
   llmserver stop 2
   ```

2. Stop all models:

   ```bash
   llmserver stop all
   ```

> **CAUTION:** Clients of the stopped models lose their connection.

### 7.2 Remove a pin

1. Remove the pin:

   ```bash
   llmserver unpin 1
   ```

   Result: llmserver can stop this model again to make space.

### 7.3 Undo a start that failed

You do not have to do an action. If a new model does not start, llmserver does these steps automatically:

1. It stops the new model.
2. It starts again the models that it stopped for the new model, on the same ports.
3. It shows an error and gives the exit status 1.

If llmserver itself stopped during the start, do the procedure in [5.2](#52-recover-after-a-crash-of-llmserver).

### 7.4 Go back to the previous engine build

1. Show the current build and the previous build:

   ```bash
   llmserver versions
   ```

2. Go back to the previous build, for example for llama.cpp:

   ```bash
   llmserver rollback llama
   ```

3. Start the models of this engine again with `--restart`.

   Result: The models use the previous build.

> **NOTE:** Do the rollback again to go forward to the newer build.

### 7.5 Go back to the specified versions

1. Delete the file of newer versions:

   ```bash
   rm ~/.llmserver/pins.local
   ```

2. Install the specified versions:

   ```bash
   llmserver upgrade
   ```

### 7.6 Repair an engine

1. Repair the runtime and the engines:

   ```bash
   llmserver doctor --fix
   ```

   Result: llmserver installs again each component that changed after installation.

### 7.7 Reset the settings

To reset one setting, type `llmserver config unset NAME`. To reset all settings:

1. Make a copy of the settings:

   ```bash
   cp ~/.llmserver/llmserver.env ~/.llmserver/llmserver.env.bak
   ```

2. Do the setup again:

   ```bash
   llmserver setup
   ```

   Result: The setup writes new settings. It keeps your engine variables.

### 7.8 Withdraw an approval

1. Withdraw the approval:

   ```bash
   llmserver trust revoke MODEL
   ```

   Result: The model stops. It cannot start until you approve it again.

2. To approve it again, type `llmserver trust approve MODEL`.

### 7.9 Delete the check records

1. Stop all models:

   ```bash
   llmserver stop all
   ```

2. Delete the records:

   ```bash
   rm -rf ~/.llmserver/trust
   ```

   Result: At the next start of each model, llmserver does the checks again.

### 7.10 Remove skills

1. Remove a skill:

   ```bash
   llmserver skills remove NAME
   ```

   Result: llmserver deletes the skill folder and its link.

2. If you changed the skill, llmserver keeps it. To remove it, add `--force`.

> **NOTE:** `llmserver uninstall` removes the skills that llmserver installed, if they did not change. It also removes the `hf` link.

### 7.11 Uninstall llmserver

> **CAUTION:** The uninstall procedure deletes the settings, the API key, the menu and the logs. It also deletes all engines in the llmserver home. Make a copy of the files that you want to keep.

1. Show the plan. This changes nothing:

   ```bash
   llmserver uninstall -n
   ```

2. Do the uninstall:

   ```bash
   llmserver uninstall
   ```

3. If llmserver asks about the downloaded models, answer the question:
   - Type `y` to delete the models that llmserver downloaded.
   - Type `n` to keep them.

4. Type `y` to uninstall.

   Result: llmserver does these steps in this sequence:
   1. It disables and deletes the systemd units `llmserver@.service` and `llmserver.service`, if they exist.
   2. It stops all models that it started.
   3. It deletes the models that it downloaded, if you selected this.
   4. It removes the skills that it installed and their links, if you did not change the skills.
   5. It deletes the llmserver home `~/.llmserver`, with the settings, the logs and the engines.
   6. It deletes the script.

Use these options if necessary:

| Option | Result |
|---|---|
| `--purge-models` | Deletes the models that llmserver downloaded, without a question |
| `--keep-config` | Keeps `llmserver.env` and `config/` (the menu, the tuning rules and the API key) in the home |
| `--keep-script` | Keeps the script |
| `--yes` | Does the uninstall without questions |

> **NOTE:** The uninstall procedure does not delete these items:
> - engines that you installed (`LLMSERVER_*_SOURCE` is `system` or a path);
> - GGUF files on your computer that you added to the menu;
> - models that other programs downloaded;
> - `/etc/llmserver_env`;
> - the `models/` folder in the home (with `LLMSERVER_MODEL_STORE=private`), if you do not delete the models;
> - a skill that you changed. The `skills/` folder stays in the home.

> **NOTE:** llmserver deletes only the model files that it downloaded. If another program uses a file, llmserver keeps the file.

> **NOTE:** llmserver deletes the llmserver home only if the home contains the llmserver marker file. It never deletes your home folder or `/`.

---

## Sources

The engine versions and checksums came from the releases of each project on 2026-10-21:

- [uv releases](https://github.com/astral-sh/uv/releases) and [Python builds](https://github.com/astral-sh/python-build-standalone/releases)
- [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases)
- [Ollama releases](https://github.com/ollama/ollama/releases)
- [vLLM on PyPI](https://pypi.org/project/vllm/) and [vLLM GPU installation](https://docs.vllm.ai/en/latest/getting_started/installation/gpu/)
- [vLLM quickstart](https://docs.vllm.ai/en/latest/getting_started/quickstart/) and [Ollama hardware support](https://docs.ollama.com/gpu)
- [Ultra-Fast LLM Inference on AMD Instinct MI355X (AMD ROCm Blogs)](https://rocm.blogs.amd.com/software-tools-optimization/megakernel-intro/README.html) (the origin of `bench`, speed estimates, speculative decoding, NUMA placement and tuning rules)

---

## License

llmserver: add your license here (for example MIT or Apache-2.0).

llmserver does not include the programs that it installs. It downloads them on your computer, and each one keeps its own license:

- Open WebUI: the Open WebUI license (BSD-3-Clause with a branding clause). You must keep its "Open WebUI" branding if more than 50 people use it in 30 days, unless you have an enterprise license.
- SearXNG: AGPL-3.0. llmserver installs it from its fixed source with its license. It adds only a version file.
- llama.cpp (MIT), vLLM (Apache-2.0), Ollama (MIT), uv (Apache-2.0 or MIT).
- Each model has its own license. `llmserver trust show MODEL` shows it.

