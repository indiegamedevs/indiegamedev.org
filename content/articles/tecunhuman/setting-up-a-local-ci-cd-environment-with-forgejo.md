---
title: "Setting Up a Local CI/CD Environment with Forgejo and Podman"
date: 2026-10-16
articles_tags: ["ci-cd", "forgejo", "git", "podman", "self-hosting"]
image: "/images/articles/tecunhuman/forgejo.svg"
image_alt: "Forgejo logo"
image_blur: false
summary: "How I use Forgejo, Forgejo Actions, Podman, and a local container registry to build and test private projects on my own computer."
---

GitHub is an excellent place to share open-source projects, and I still use it for my public repositories. Private projects create a different set of tradeoffs. GitHub-hosted Actions for private repositories use a monthly allowance of minutes and artifact storage, then require paid usage after that allowance is exhausted. A hosted service also becomes unavailable when the service is down or my internet connection is unavailable.

For projects that I want to keep private or build frequently, I wanted a CI/CD environment that runs on hardware I already own. I also wanted to keep using a familiar Git workflow and Actions-style YAML files rather than inventing a collection of machine-specific build scripts.

I built that environment with [Forgejo](https://forgejo.org/), [Forgejo Actions](https://forgejo.org/docs/latest/user/actions/overview/), and rootless [Podman](https://podman.io/). Forgejo hosts the repositories and container images, while a local runner builds and tests each change. The complete setup is available in these repositories:

- [tecunhuman/workspace](https://github.com/tecunhuman/workspace) ties the projects together as Git submodules and provides the top-level backup commands.
- [tecunhuman/forge](https://github.com/tecunhuman/forge) runs Forgejo and a Forgejo Actions runner with Podman, plus a separate Docker-in-Docker daemon for CI jobs that build images.
- [tecunhuman/build-containers](https://github.com/tecunhuman/build-containers) contains the build environments I use for cross-platform Godot projects.
- [tecunhuman/project-example](https://github.com/tecunhuman/project-example) is a small C project that demonstrates Linux and Windows builds in the local CI environment.

This article covers how I bring up the Forgejo server, register the runner, create repositories, publish build containers, and run the example workflows.

## Why Run CI/CD Locally?

The main benefit is control. My repositories, runner capacity, build history, container images, and artifacts live on storage I manage. Builds consume my own CPU time instead of a hosted minute allowance, and I decide when to upgrade the server or runner.

The setup also keeps working when the wider internet is unavailable, provided the workflow dependencies and container images have already been downloaded. That qualification matters: a first run may still need to pull Forgejo images, build-image dependencies, and reusable actions such as `actions/checkout`. A local forge removes the hosted service from the normal development loop, but it does not automatically mirror every external dependency.

There are costs as well. I am responsible for updates, security, storage, backups, electricity, and recovering the service if the host fails. For public projects, GitHub's free hosted runners remain very convenient. I see the local system as a complement to GitHub rather than a universal replacement.

## Prerequisites

I run this setup on a Linux computer with:

- Git
- Podman
- `podman-compose`
- The `podman-docker` compatibility package
- Make
- `rsync` for copying backups to another machine

I use rootless Podman, so the containers run as my normal user rather than through a root-owned Docker daemon. I still use Docker commands throughout the workspace: the `podman-docker` package provides a compatible `docker` command, and `podman-compose` handles Compose files. On Ubuntu, I install the required packages with:

```bash
sudo apt install podman podman-compose podman-docker make git rsync
```

The existing Forgejo Makefile and workflows can therefore keep using Docker commands even though Podman handles the containers on the host. I verify the compatibility interface with:

```bash
docker --version
docker compose version
```

The commands should report that the Docker CLI is being emulated by Podman. There is no need to add my user to a `docker` group for this rootless setup.

## Running the Setup on Windows with WSL 2

I can also run the same environment on Windows through [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/about). WSL 2 provides the Linux kernel features needed by Podman, so the Forgejo server, runner, and build containers can remain Linux-based. I run the entire stack inside an Ubuntu WSL distribution rather than mixing Windows and Linux container tools.

From an administrator PowerShell window, I install WSL and Ubuntu:

```powershell
wsl --install -d Ubuntu
```

Windows may require a restart. After Ubuntu has completed its first-time user setup, I verify that it is using WSL 2:

```powershell
wsl --list --verbose
```

If the Ubuntu entry reports version 1, I convert it from PowerShell:

```powershell
wsl --set-version Ubuntu 2
```

I then open the Ubuntu terminal and install the same packages used by the native Linux setup:

```bash
sudo apt update
sudo apt install podman podman-compose podman-docker make git rsync
docker --version
docker compose version
```

I clone the workspace under my Linux home directory, such as `~/workspace`, and run all of the remaining commands from the Ubuntu terminal. I avoid placing an active repository below `/mnt/c`. Microsoft recommends keeping projects used by Linux command-line tools in the WSL filesystem because builds and Git operations across the Windows-mounted filesystem can be considerably slower. Windows applications can still browse the files at a path such as:

```text
\\wsl.localhost\Ubuntu\home\my-user\workspace
```

The `forge.lan` name must resolve in both environments. Inside Ubuntu, I add the same entry described later in this article to `/etc/hosts`:

```text
127.0.0.1  forge.lan
```

To use that name in a Windows browser, I also open this file as an administrator:

```text
C:\Windows\System32\drivers\etc\hosts
```

I add the same line there:

```text
127.0.0.1  forge.lan
```

[WSL normally forwards services running in Linux to Windows localhost](https://learn.microsoft.com/en-us/windows/wsl/networking), so after starting the stack from Ubuntu I can open [http://forge.lan:3000](http://forge.lan:3000) in a Windows browser. If localhost forwarding has been disabled or changed by a custom WSL network configuration, I first test `http://localhost:3000` and review the WSL networking settings.

The normal workflow remains unchanged inside Ubuntu:

```bash
cd ~/workspace/forge
make up
docker compose ps
```

WSL must be running for Forgejo and its runner to remain available. After `wsl --shutdown`, I reopen Ubuntu and run `make up` again when I want to restart the local environment.

## Cloning the Workspace

The top-level repository uses Git submodules, so I clone it recursively:

```bash
git clone --recurse-submodules https://github.com/tecunhuman/workspace.git
cd workspace
```

If I already cloned it without the submodules, I initialize them separately:

```bash
git submodule update --init --recursive
```

The resulting layout is:

```text
workspace/
├── build-containers/
├── forge/
└── project-example/
```

Each directory remains an independent Git repository. The workspace repository records which revision of each project belongs to the complete setup.

## Configuring the Local Forge

I start in the `forge` directory and create its local configuration:

```bash
cd forge
cp .env.example .env
```

The example contains three settings:

```dotenv
FORGE_HOST=forge.lan
FORGE_DATA=~/.local/share/forge
RUNNER_CAPACITY=4
```

`FORGE_HOST` is the name used by my browser, Git, the runner, workflow containers, and the container registry. `FORGE_DATA` is where Forgejo and runner state live. `RUNNER_CAPACITY` controls how many jobs the runner may execute concurrently across the forge.

I add the local name to the host computer's `/etc/hosts` file:

```text
127.0.0.1  forge.lan
```

Using the same hostname everywhere is important. On the host, `forge.lan` resolves to the loopback interface. Inside the Compose network, the same name is an alias for the Forgejo container. It also means that the browser hostname matches Forgejo's session cookie and generated URLs.

The Compose configuration publishes only these host ports:

```text
127.0.0.1:3000  Forgejo web interface and container registry
127.0.0.1:2222  Forgejo SSH server
```

Binding them to `127.0.0.1` keeps the service local to the computer. If I later want access from another device, I would add a deliberate network or SSH-tunnel configuration rather than exposing it accidentally.

## Starting Forgejo

The repository's Makefile creates the data directories and gives the runner container permission to use its directory:

```bash
make init
make server
```

I then open [http://forge.lan:3000](http://forge.lan:3000) and complete Forgejo's initial setup. The first account I create becomes the administrator.

The server is started separately at this stage because the runner must be configured and registered against a working Forgejo instance before its daemon can connect.

## Registering the Actions Runner

Forgejo stores workflows, schedules jobs, and displays their results, but it does not execute the jobs itself. That work belongs to [Forgejo Runner](https://forgejo.org/docs/latest/admin/actions/).

I generate a runner configuration and register it with the local server:

```bash
make runner-config
make register
```

The Make targets use the Docker-compatible command supplied by `podman-docker`, so I do not need to change the repository's existing commands. Podman creates and manages the containers underneath that interface. The registration command obtains a runner token from the local Forgejo container. It registers a runner named `local` with this label:

```text
docker:docker://node:20-bookworm
```

The word `docker` here is the runner label and container execution type, not a requirement that the host use Docker Engine. A workflow using `runs-on: docker` can therefore be picked up by this runner. The Node image is also useful because many reusable Actions require a Node runtime.

After registration, I start the complete stack:

```bash
make up
```

I can inspect it with:

```bash
docker compose ps
make logs
```

The stack has three services:

- `server` runs Forgejo.
- `runner` receives and executes Forgejo Actions jobs.
- `docker-in-docker` gives workflows a separate Docker-compatible daemon for building container images.

Although Podman runs the services on the host, the current Compose stack uses a privileged `docker:dind` container as the runner's isolated build daemon. Build jobs do not receive the host's Podman socket. That prevents an ordinary workflow from directly inspecting or changing the Forgejo server container. Jobs sharing the inner daemon can still affect one another, so I only allow trusted users and trusted workflows on this instance. The [Forgejo container access documentation](https://forgejo.org/docs/latest/admin/actions/docker-access/) explains the security tradeoffs in more detail.

Rootless Podman maps container user IDs into a user namespace. If I need to inspect or edit the runner configuration directly, I enter that namespace first:

```bash
podman unshare nano ~/.local/share/forge/runner/runner-config.yml
```

That path matches the example `FORGE_DATA` value. I change it if I selected a different data directory.

## Creating the Organization and Repositories

I can create the organization and repositories in the web interface, but the `forge` repository also includes a small API tool for repeatable command-line provisioning.

First I create an API token for the administrator and export it only for the current shell session:

```bash
export TOKEN=$(./forge-api.py token admin)
```

I then create the organization and repositories:

```bash
./forge-api.py org create tecunhuman --description "Tecunhuman projects"
./forge-api.py repo create tecunhuman/workspace --description "Development infrastructure and example projects"
./forge-api.py repo create tecunhuman/forge --description "Local Forgejo server and Actions runner"
./forge-api.py repo create tecunhuman/build-containers --description "Cross-platform Godot build containers"
./forge-api.py repo create tecunhuman/project-example --description "Example project for the local CI environment"
```

I can add `--private` to any `repo create` command when the source should not be visible to other Forgejo users.

For example, I add a local remote to the workspace repository like this:

```bash
git remote add forgejo ssh://git@localhost:2222/tecunhuman/workspace.git
git push -u forgejo main
```

The public GitHub remote and private local remote can coexist. I can push selected revisions to GitHub for publication while using Forgejo for my normal build loop.

## Building and Publishing CI Images

The example project has two containerized toolchains:

- Ubuntu builds a native Linux executable.
- MinGW cross-compiles a Windows executable from Linux.

Before running normal builds, I publish those build environments to Forgejo's local [container registry](https://forgejo.org/docs/latest/user/packages/container/). The workflow is stored at:

```text
.forgejo/workflows/build_images.yml
```

It builds images with names such as:

```text
forge.lan:3000/tecunhuman/project-example-ubuntu:v1
forge.lan:3000/tecunhuman/project-example-mingw:v1
```

The workflow needs a Forgejo access token that can publish packages. I create that token in Forgejo, then add it to the `project-example` repository as an Actions secret named `REGISTRY_TOKEN`.

From the repository's Actions page, I run the `docker_builds` workflow manually. It can also run from its monthly schedule. The job uses the Docker-compatible CLI inside its job container, logs into the local registry, builds both images with the isolated daemon, and pushes `v1` and `latest` tags.

## Building and Testing the Example Project

The normal workflow is stored at:

```text
.forgejo/workflows/builds.yml
```

It runs on pushes and pull requests targeting `main`, and it can also be started manually. The workflow has two independent jobs.

The Linux job uses the Ubuntu image, checks out the repository, builds the program, and runs it:

```yaml
- name: Build project-example
  run: ./platform/ubuntu/build.sh

- name: Test project-example
  run: ./build/linux/project-example | grep "hello from project-example"
```

The Windows job uses the MinGW image, cross-compiles the program, and verifies that the expected executable exists:

```yaml
- name: Build project-example
  run: ./platform/mingw/build.sh

- name: Check project-example
  run: ls -la build/windows/project-example.exe
```

Both jobs authenticate to the private local registry with the same `REGISTRY_TOKEN` secret. Pushing a commit to the local `main` branch now triggers the complete build:

```bash
git push forgejo main
```

The same builds can be run without Forgejo when I am developing the project locally:

```bash
make ubuntu
make mingw
```

That parity is useful. The local command and the CI job use the same build scripts and container definitions, which reduces the difference between a build that works at my terminal and one that runs in automation.

## Backing Up the Forge

Self-hosting makes backups my responsibility. The workspace has two layers of backup.

Inside `forge`, this command asks Forgejo to create a dump and creates a Git bundle for every non-empty repository:

```bash
make backup
```

The dumps are written to `forge/archives/`, and the bundles are written below `forge/bundles/`. A Git bundle is especially useful because it can be cloned independently:

```bash
git clone forge/bundles/tecunhuman/project-example.bundle
```

At the workspace root, I copy the example backup configuration and choose a destination on another machine:

```bash
cd ..
cp .env.example .env
```

For example:

```dotenv
BACKUP_TARGET=developer@backup-host:/srv/backups/workspace
BACKUP_DELETE=0
SKIP_FORGE_DUMP=0
```

Before copying anything, I check the proposed transfer with an `rsync` dry run:

```bash
make backup-check
```

Then I run the real backup:

```bash
make backup
```

The ordinary backup does not delete files from the destination. `make backup-delete` mirrors deletions too, so I use it only when that is intentional.

The backup destination should be on a different disk or computer. A local dump stored beside the live Forgejo data is useful for recovery, but it does not protect against losing the host drive.

## What This Setup Gives Me

With the stack running, a normal change follows a familiar path:

1. I commit code to a local Git repository.
2. I push it to Forgejo.
3. Forgejo sends the workflow to the local runner.
4. The runner starts isolated build containers.
5. The project is built and tested for Linux and Windows.
6. Forgejo records the result beside the commit or pull request.

This gives me the parts of hosted development that I use most—Git hosting, pull requests, Actions-style workflows, a container registry, build logs, and status results—on my own hardware. There is no hosted Actions minute allowance to watch, and ordinary development can continue when GitHub or my internet connection is unavailable.

It is not maintenance-free, and it is not automatically disconnected from every online dependency. What it gives me is a development environment whose availability, capacity, privacy, and upgrade schedule are under my control. I can still mirror public work to GitHub, but GitHub no longer has to be the only place where my projects can be built.
