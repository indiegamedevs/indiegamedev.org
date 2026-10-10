---
title: "Running a Local LLM with HyperQwen"
date: 2026-10-10
articles_tags: ["ai", "codex", "linux", "local-llm", "ubuntu"]
image: "/images/articles/tecunhuman/qwen-logo.svg"
image_alt: "Qwen logo"
image_blur: false
summary: "How I run Qwen locally with HyperQwen on an Ubuntu computer with an NVIDIA RTX 3090 and connect to it remotely from Codex."
---

I have always been passionate about open-source software because it lets me retain control over my development environment. I can choose my tools, inspect how they work, modify them, and keep a known version available when a project depends on it.

As AI keeps growing, it becomes more difficult to ignore. Using it, however, should not mean giving up the freedoms I value in open-source software. With a hosted AI service, the model can disappear or change without my involvement. Prices, usage limits, features, and terms can change as well. Anything I send to the model has to leave my network, and the tool becomes unavailable when the service is down, my internet connection is unavailable, or I exhaust my allotted credits.

Running AI locally gives me control over that part of the toolchain. I decide which model and version to run, how much context to give it, when to upgrade it, and who can connect to it. My prompts, source code, and tool output stay on computers I control. Once the model has been downloaded, I can continue using it without a subscription, an API quota, or permission from the company operating a remote server.

Hosted AI is still ahead of local AI in some areas, but the gap keeps shrinking. Video generation is a good example. When Sora appeared, that kind of capability seemed firmly tied to large hosted services. Not long afterward, models such as [MiniMax H3](https://github.com/MiniMax-AI/MiniMax-H3) made it possible to generate video and audio with downloadable models running on local hardware. Local models require capable hardware, electricity, storage, and maintenance, but their rapid progress makes dependence on hosted AI feel less inevitable. In the same way that open-source software protects my ability to keep building when a vendor changes direction, a local model gives me an AI tool that I can continue operating for myself.

I have an Ubuntu computer with an NVIDIA RTX 3090 that I can dedicate to running AI models. The card's 24 GB of VRAM makes it a good fit for [HyperQwen](https://github.com/syv-ai/HyperQwen), a project for serving optimized Qwen models on consumer GPUs. HyperQwen provides an OpenAI-compatible API and includes separate configurations for interactive use and concurrent batch requests.

This article covers the setup I use, including the changes I made for a larger context window and vision support. It also shows how I reach the server securely from another computer and use the model with the Codex CLI.

## Prerequisites

Before installing HyperQwen, I already had the following working on the Ubuntu server:

- An NVIDIA RTX 3090 with a functioning NVIDIA driver
- Docker with the Compose plugin
- The NVIDIA Container Toolkit, configured so Docker containers can use the GPU
- Git and Make

It is worth verifying GPU access before downloading the model:

```bash
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

The exact CUDA container tag can change over time. The important result is that `nvidia-smi` works both on the host and inside a Docker container.

The initial HyperQwen startup downloads a large container image and the model files, then prepares the model for serving. Make sure the server has plenty of free disk space before starting.

## Installing HyperQwen

I cloned the HyperQwen repository and created my local environment file from the example:

```bash
git clone https://github.com/syv-ai/HyperQwen.git
cd HyperQwen
cp .env.example .env
make
```

Running `make` by itself prints the available shortcuts. The commands I use most often are:

```bash
make up-single
make up-batch
make logs
make ps
make down
```

The Make targets delegate to Docker Compose, so I can use short commands without having to remember the profile syntax.

## My Environment Settings

I edited `.env` and changed or added these values:

```dotenv
CTX=huge
VLLM_WSL2_ENABLE_PIN_MEMORY=0
VISION=1
VISION_OFFLOAD=1
```

`CTX=huge` selects HyperQwen's huge-context configuration. I use it with the interactive single-user server to give Codex a context window of 245,760 tokens. This gives the model more space to work with files, tool output, and conversation history before Codex has to compact the context.

I run native Ubuntu rather than Docker Desktop under WSL2, so my configuration sets `VLLM_WSL2_ENABLE_PIN_MEMORY` to `0`. HyperQwen's example enables it because the setting is required on WSL2. According to the project's comments, leaving it enabled on native Linux is also harmless, so changing it is not required for every Ubuntu installation.

The `VISION` settings enable the vision path and its offloading behavior. They are part of my configuration rather than a requirement for running the text model.

The `.env` file is ignored by Git, which makes it the appropriate place for machine-specific tuning and API credentials.

## Starting the Interactive Server

For normal coding sessions, I start the single-user configuration:

```bash
make up-single
```

This mode is designed for one person or a small number of interactive users. It favors the latency of an individual request rather than total throughput across many simultaneous requests.

The first startup takes much longer than later starts because HyperQwen must download and prepare its files and compile several components. I follow its progress with:

```bash
make logs
```

I can check the container and health endpoint separately with:

```bash
make ps
```

By default, the server listens on port `18020` and Docker publishes it only on `127.0.0.1`. Keeping that default means the API is not exposed directly to the local network.

## Connecting from Another Computer

I normally run Codex on a different computer than the one with the RTX 3090. I connect the two with SSH port forwarding:

```bash
ssh -N \
  -o ExitOnForwardFailure=yes \
  -L 18020:localhost:18020 \
  gamedev
```

`gamedev` is the SSH hostname for my Ubuntu server. This command maps port `18020` on my client computer to port `18020` on the server.

The `-N` option opens the connection without starting a remote shell. `ExitOnForwardFailure=yes` makes SSH report a failure instead of leaving me with a connection that does not have a working tunnel.

I leave this SSH process running while using the model. HyperQwen is then available to software on the client at:

```text
http://localhost:18020
```

This lets HyperQwen remain bound to the server's loopback interface. I do not need to expose an unauthenticated inference port to the network because SSH provides the remote access layer.

## Using HyperQwen with Codex

On the client computer, I point the Ollama CLI at the forwarded port:

```bash
export OLLAMA_HOST=http://localhost:18020
```

I then use Ollama's Codex integration to start the Codex CLI with the model served by HyperQwen:

```bash
ollama launch codex \
  --model qwen3.8-27b \
  -- \
  -c model_context_window=245760 \
  -c model_auto_compact_token_limit=230000 \
  -c show_raw_agent_reasoning=true \
  -c model_reasoning_effort=none \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true
```

The `--` separates the arguments for `ollama launch` from the arguments passed to Codex.

The two context settings are especially important in this configuration. `model_context_window` tells Codex how much context is available, while `model_auto_compact_token_limit` tells it to compact the conversation before reaching the server's limit. These settings do not increase the capacity of the model server; they describe the HyperQwen configuration that is already running.

I enable `workspace-write` so Codex can edit files in the project from which I launched it. I also allow network access for commands running inside that sandbox. That is a deliberate permission choice: it is useful when a project needs to download dependencies or reach external services, but it gives commands run by the agent more access than an offline sandbox would.

`show_raw_agent_reasoning=true` is useful while experimenting with the local model because it lets me see reasoning content returned by the server. I set `model_reasoning_effort=none` so Codex does not request a separate reasoning-effort level from this model.

## Switching to Batch Mode

HyperQwen also has a batch configuration for API backends and multiple concurrent requests. I stop the current container before switching modes:

```bash
make down
make up-batch
```

One GPU should run only one of these modes at a time. Single-user mode is my normal choice for an interactive Codex session; batch mode is useful when aggregate throughput matters more than the speed of one request.

When using batch mode, I give Codex a smaller context window and compaction threshold:

```bash
ollama launch codex \
  --model qwen3.8-27b \
  -- \
  -c model_context_window=150000 \
  -c model_auto_compact_token_limit=145000 \
  -c show_raw_agent_reasoning=true \
  -c model_reasoning_effort=none \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true
```

The lower automatic-compaction value leaves some room for Codex to finish compacting before the request reaches the model's context ceiling.

## The Result

This arrangement keeps the expensive part of the setup on the Ubuntu workstation. HyperQwen runs Qwen3.8-27B on the RTX 3090 and listens only on the server's local interface. On my client computer, the Ollama launcher and Codex connect to port `18020`, and the SSH tunnel securely forwards that connection to HyperQwen.

Using the model from the client computer feels much like using any other Codex provider. The model, cache, and GPU workload remain on the server, while Codex works with the files in my local project. I get a private coding setup that makes practical use of the RTX 3090's 24 GB of VRAM, supports a very large context window, and remains available from my other computers without exposing the model API directly to the network.

[Originally published on Patreon](https://www.patreon.com/tecunhuman/posts/running-local-172001390)
